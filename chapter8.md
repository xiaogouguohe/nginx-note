# 第8章 nginx基础架构

- 本章的写作目的
  - 对Nginx的设计思路做概括性说明，了解Nginx的设计原则（8.1和8.2）
  - 从具体的代码框架入手，讨论Nginx如何启动、运行和退出，设计具体实现细节，如master进程如何管理worker进程、每个模块如何加载到进程中（8.38~8.6）

## 8.1 Web服务器设计中的关键约束

###

## 8.2 Nginx的架构设计

- 在8.1列出的7个关键点上提升Nginx的能力

### 8.2.1 优秀的模块化设计

- 在Nginx中，除了少量核心代码，其它皆为模块，模块化设计具有以下特点

  - 高度抽象的模块接口

    - 所有模块都遵循ngx_module_t接口设计规范
    - ngx_module_t结构首次出现在3.4，也可以看图8-1

  - 模块接口简单，灵活

    - ngx_module_t只涉及模块的初始化、退出和对配置项的处理，对应到具体的方法就是图8-1的7个方法，只定义了这些方法
    - 在8.4~8.6中，可以看到以上7个回调方法何时会被调用
    - ngx_command_t类型的commands数组则指定了模块处理配置项的方法（第4章）
    - ngx_module_t中的ctx成员是void*类型，可以指向任意类型的模块，见图8-1，可以是ngx_core_module_t、ngx_http_module_t等等类型的结构体

  - 配置模块的设计

    - ngx_module_t有一个type成员（见3.4），类型是ngx_uint_t，说明模块类型，和ctx配合使用
      - type成员取值范围为：
        - NGX_HTTP_MODULE
        - NGX_CORE_MODULE
        - NGX_CONF_MODULE
        - NGX_EVENT_MODULE
        - NGXX_MAIL_MODULE
      - 模块类型分为几种，其中有一种是配置模块类型（此时type取值NGX_CONF_MODULE），这个类型仅有一个模块ngx_conf_module，它指导所有模块以配置项为核心来提供功能

  - 核心模块接口的简单化

    - 模块类型当中有一种是核心模块类型（此时type取值NGX_CORE_MODULE），有6个具体模块，分别为ngx_core_module、ngx_errlog_module、ngx_events_module、ngx_openssl_module、ngx_http_module、ngx_mail_module，其它框架代码只关注如何调用6个核心模块

    - 核心模块的结构（ctx指向的结构）

      ```c
      typedef struct {
          // 核心模块名称
          ngx_str_t name;
          // 解析配置项前，Nginx框架会调用create_conf方法,
          // create_conf创建存储配置项的数据结构，创建出来的是返回值指向的，还是cycle？？？
          // 在读取nginx.conf配置文件时，会根据模块中的ngx_command_t把解析出的配置项存放到这个数据结构中，什么时候解析配置项？？？
          void *(*create_conf)(ngx_cycle_t *cycle);
          // 解析配置项完成后，Nginx框架会调用init_conf方法
          // init_conf会使用解析出的配置项，来初始化核心模块功能
          char *(*init_conf)(ngx_cycle_t *cycle, void *conf);
      } ngx_core_module_t;
      ```

    - 除了上述代码提到的以外，Nginx不会约束核心模块的接口和功能

    - 这种设计使得每个核心模块都可以自由定义全新的模块类型，例如核心模块ngx_http_module就可以定义和管理类型为NGX_HTTP_MODULE的模块（也就是http模块）...通过什么数据结构来定义和管理http模块，按照之前的说法，应该是能找到一个ngx_module_t的结构，如何找到的？？？

  - 多层次、多类别的模块设计

    - Nginx有五大类型的模块：核心模块、配置模块、事件模块、HTTP模块、mail模块，它们具备相同的ngx_module_t接口（也就是说都是由ngx_module_t的成员ctx指向的），但是它们在请求流程中的层次不同
    
      - 配置模块和核心模块是由Nginx的框架代码所定义的（也就是说函数指针都已经实现？？？）
      - 其它三种模块都不会和框架产生直接联系，而是在核心模块中各有一个模块作为自己的“代言人”，并且在同类模块中有1个模块实现核心业务和管理功能，见图8-2
        - 例如，事件模块是由核心模块ngx_events_module定义，并且由事件模块ngx_event_core_module模块负责加载操作（加载是指给成员赋值等等操作？？？）
        - 和事件模块不同，HTTP模块的加载是由核心模块ngx_http_module负责的，而非同类模块ngx_http_core_module，但是ngx_http_core_module负责选用某个具体的HTTP模块对请求进行处理
    
      - 配置模块和核心模块是其它模块的基础；事件模块是HTTP模块和mail模块的基础（8.2.2）；HTTP模块和mail模块的地位相似，都更关注应用层面
    

### 8.2.2 事件驱动架构

#### 8.2.2.1 传统的服务器处理事件的模型

- 简化的模型见图8-3
- 建立连接就开一个进程（线程），连接关闭就释放这个进程，在处于连接状态的过程中会一直保留这个进程
- 从“事件驱动”的角度去看这个模型，每个这样的进程都是事件消费者
- 这样的劣势就是，整个连接状态的过程中，可能只有少数时候有事件，也就是说大部分的时间，进程消费者都在等待事件，造成浪费

#### 8.2.2.2 Nginx处理事件的模型

- 见图8-4
- 只有事件的收集和分发需要维持一个进程（通过一个进程来收集和分发事件），把这个进程称为收集和分发进程
- 从“事件驱动”的角度去看这个模型，这里的事件消费者是各个模块，有事件到来的时候，收集和分发进程把事件分发给消费者模块，只有这个时候才会起一个进程（是这样吗？？？）

- 这样的优势就是，不用一直维护那么多进程等待事件到来，而是在事件来的时候再起一个进程，而且也能够保证及时响应（因为一旦有事件来，就可以及时通知事件消费者来处理）
- 但这样也有个弊端，就是每个事件消费者都不能有阻塞行为，否则会长期占用收集和分发进程，导致其他事件无法及时得到响应，尤其是事件消费者不能让进程休眠或等待？？？

### 8.2.3 请求的多阶段异步处理

- 请求的多阶段异步处理，是把一个请求的处理过程，按照事件的触发方式划分为多个阶段，而这只能基于事件驱动架构实现
- 一个如何划分请求阶段的例子
  - 表8-1
- 请求被划分为多个阶段，每个阶段的处理只能等待内核的通知，当下一次事件出现时，epoll等事件分发器会获取到通知，再继续调用事件消费者处理请求
- 划分请求阶段的一些原则，找到阻塞方法（阻塞代码段），按照以下方式来划分阶段
  - 将阻塞进程的方法按照相关的触发事件分解为两个阶段
    - 第一个阶段调用方法，第二个阶段接收调用方法的结果
    - 以send发送数据给用户为例，如果使用阻塞socket句柄，那么进程在发出数据后就休眠，直到成功发送数据才被唤醒
    - 可以使用非阻塞socket句柄，这样调用send之后，进程不休眠，而是把socket句柄加入到事件收集器，（然后进程可能结束），等待相应的事件（这里是send成功）触发下一阶段（再起一个进程）
  - 将阻塞方法调用按照时间分解为多个阶段
    - 有些时候第二阶段的事件没法被收集
    - 例如读取文件的调用，读10MB的文件，读取的文件可能在磁盘上不是连续的，而磁盘寻址比较耗时，因此进程一般会休眠
    - Nginx事件模块在没打开异步I/O的时候，不支持上面提到的分两个阶段
    - 可以每次读10KB，触发下一个阶段（读下一个10KB）的事件可以是网络事件（客户端请求下一个10KB），也可以是定时器
  - 在“无所事事”且必须等待系统响应，从而导致进程空转时，用定时器划分阶段
    - 忙等待，怎么解决？？？
  - 如果阻塞方法完全无法继续划分，则使用独立的进程执行这个方法

### 8.2.4 管理进程、多工作进程设计

- 图8-5
- 设计的优点
  - 利用多核系统的并发处理能力
  - 负载均衡
    - 进程间通信来实现负载均衡
  - 管理进程监控工作进程，管理其行为

### 8.2.5 平台无关的代码实现

- Nginx尽量减少与操作系统平台相关的代码，实现可移植性

### 8.2.6 内存池的设计

- 最大优点在于把多次向系统申请和释放内存的操作整合成一次，减少CPU资源的消耗，减少内存碎片
- 每个请求都会有一个这样的内存池，在请求结束时会销毁内存池，把分配的内存全部归还给操作系统，也就是说申请内存不用考虑它的释放

### 8.2.7 使用统一管道过滤器模式的HTTP模块

###

### 8.2.8 其它一些用户模块

###

## 8.3 Nginx框架中的核心结构体ngx_cycle_t

- 每个进程都拥有唯一一个ngx_cycle_t结构体

### 8.3.1 ngx_listening_t结构体

- ngx_cycle_t对象中有一个动态数组成员为listening，每个数组元素都是ngx_listening_t结构体，每个ngx_listening_t结构体代表Nginx服务器监听的一个端口
  - 8.3.2中的一些方法会使用ngx_listening_t结构体来处理要监听的端口
  - 8.4中的master进程、worker进程等进程如何监听同一个TCP端口（fork出来的紫禁城自然共享打开的端口）
  - 更多ngx_listening_t的介绍见第9章
  
- ngx_listening_t的成员

  - 见书上代码

  - ngx_connection_handler_pt类型的handler成员，表示在这个监听端口上成功建立新的TCP连接后，会回调handler方法，定义如下：

    ```c
    typedef void (*ngx_connection_handler_pt) (ngx_connection_t *c)
    ```

    接收一个ngx_connection_t连接参数，许多事件消费模块会自定义handler方法

  - handler一般做什么事情？ngx_connection_t参数一般做什么？

### 8.3.2 ngx_cycle_t结构体

- ngx_cycle_t成员
  - 代码见书
  - 和配置文件相关的成员

### 8.3.3 ngx_cycle_t支持的方法

- 支持的方法见表8-2

## 8.4 Nginx启动时框架的处理流程

- 本节极少Nginx在启动时框架做了什么，不涉及Nginx具体某个模块做的工作，主要是调用表8.2列出来的方法

- Nginx框架在启动时对的工作流程见图8-6

  1. 根据命令行得到配置文件路径

     - 预先创建一个临时的ngx_cycle_t类型变量，用它的成员存储配置文件路径，相关的成员如下：

       ```c
       typedef struct ngx_cycle_s ngx_cycle_t;
       
       struct ngx_cycle_s {
           // 配置文件相对于安装目录的路径名称，nginx.conf
           ngx_str_t conf_file;
           // Nginx处理配置文件时需要特殊处理的命令行参数，一般是-g选项携带的参数
           ngx_str conf_param;
           // Nginx配置文件所在目录的路径
           ngx_str_t conf_prefix;
           // Nginx安装目录的路径
           ngx_str_t prefix;
       }
       ```

     - 然后调用表8-2中的ngx_process_options方法来设置配置文件路径等参数cycle

       ```c
       // cycle是刚分配的ngx_cycle_t结构体指针，仅用于传递配置文件路径信息（也就是只用到上面提到的成员），
       // 通过这些成员来生成完整的配置文件路径
       // cycle可能是传入传出参数，根据cycle的一些成员，生成另一些成员？？？
       ngx_int_t ngx_process_options(ngx_cycle_t *cycle)
       ```

  2. 图8-6的第2步实际上是在调用表8-2的ngx_add_inherited_sockets方法

     - 通过不重启服务的方式来升级（1.9）时，不重启master就可以启动新版本的Nginx程序

     - 如何不重启master？通过execve系统调用（先fork出子进程再调用exec运行新程序）

     - 这样，旧版本的master进程通过环境变量来传递一些信息，告诉新版本的master进程这是在平滑升级，环境变量如何存储信息？？？需要存储哪些信息？？？

     - 新版本的master进程通过ngx_add_inherited_sockets方法从环境变量里读取平滑升级的相关信息，并且对旧版本服务监听的句柄做继承处理，如何读取环境变量？？？

     - ngx_add_inherited_sockets方法的声明

       ```c
       // 不重启服务升级Nginx时，老Nginx进程会通过环境变量"NGINX"来传递要打开的监听端口，如何把这些信息放到环境变量里？？？
       // 新Nginx进程会通过ngx_add_inherited_sockets方法来使用已经打开的TCP监听端口，如何使用？？？
       ngx_int_t ngx_add_inherited_sockets(ngx_cyce_t *cycle)
       ```

  3. 第3~8步，都是在ngx_init_cycle方法中执行

     - ngx_init_cycle方法

       ```c
       // old_cycle表示临时的ngx_cycle_t指针，仅用来传递和配置文件相关的参数
       // 返回初始化成功的完整ngx_cycle_t结构体，不能复用吗？？？
       // 该函数负责以下工作：
       // 初始化ngx_cycle_t中的数据结构，
       // 解析配置文件，
       // 加载所有模块，
       // 打开监听端口（第7步），
       // 初始化进程间通信方式（第6步）
       ngx_cycle_t *ngx_init_cycle(ngx_cycle_t *old_cycle)
       ```

     - 第3步，调用所有核心模块的create_conf方法， 生成存放配置项的结构体
       - 每个模块都必须有相应的数据结构来存储配置项
       - 框架只关心核心模块，所以只调用核心模块的create_conf方法（也只有核心模块有这个方法）
       - 非核心模块大多从属于一个核心模块
         - 例如每个HTTP模块都由ngx_http_module管理（见图8-2）
         - ngx_http_module在解析自己感兴趣的"http"配置项时，会调用所有HTTP模块约定的方法create_xxx_conf来创建存储配置项的结构体（第4章）

  4. 对所有核心模块解析配置项（4.3.1和图4-1）

  5. 调用所有核心模块的init_conf方法，让所有核心模块在解析完配置项后可以做综合性处理（？？？）

  6. 创建或打开相关的目录和文件，同时ngx_cycle_t结构体的shared_memory链表会初始化用于进程间通信的共享内存

     - 目录和文件，和ngx_cycle_t结构体的pathes和open_files这两个成员有关
     - 有些模块（如缓存模块）已经添加了这些文件和目录

  7. 为ngx_cycle_t的listening数组的每一个ngx_listening_t元素设置socket句柄并监听端口

     - 调用表8-2中的ngx_open_listening_sockets方法

       ```c
       ngx_int ngx_open_listening_sockets(ngx_cycle_t *cycle)
       ```

  8. 调用所有模块的init_module方法，接下来会根据配置的Nginx运行模式决定如何工作

  9. 如果为单进程工作模式会调用ngx_single_process_cycle方法进入单线程工作模式

  10. 调用所有模块的init_process方法。至此启动工作全部完成，进入单进程工作模，也就是worker（8.5）、master（8.6）进程工作循环的结合体

  11. 如果为master、worker工作模式，在启动worker子进程、cache manager子进程、cache loader子进程后 ，进入8.6的工作状态。至此master进程启动流程执行完毕

  12. 由master进程按照配置worker的数目启动worker进程

      - 调用ngx_start_worker_processes方法

        ```c
        // cycle是当前进程的ngx_cycle_t指针
        // n是启动子进程的个数
        // type是启动方式，见8，6
        void ngx_start_worker_processes(ngx_cycle_t *cycle, ngx_int_n, ngx_int_t type)
        ```

  13. 调用所有模块的init_process方法。worker进程启动工作至此完成，接下来进入循环处理事件流程（8.5）

      - ngx_worker_process_cycle方法

        ```c
        // data一般还未开始使用这个参数，因此一般为NULL
        // 进入worker进程工作的循环（不断获取事件）
        void ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
        ```

  14. master进程根据各模块的初始化情况来决定是否启动cache manager进程
      - ###
  15. cache loader
      - ·###
  16. 关闭只有worker进程才需要监听的端口，为什么要关闭？？？

## 8.5 worker进程是如何工作的

- master采用信号来通知worker停止服务或更换日志文件

  - 信号见《Unix环境高级编程第10章》，尚存一些疑问

  - 四个相关的信号

    - 表8-3
    - 通过四个全局标志位来标识这四个信号

  - 处理信号的方法

    ```c
    void ngx_signal_handler(int signo)
    ```

- worker进程的生命周期

  - ngx_worker_process_cycle就是整个生命周期
  - 图8-7

## 8.6 master进程是如何工作的

- 和worker类似，通过7个标志位（信号）来决定ngx_master_process_cycle方法的运行

  - 表8-4
  - 从哪里接收信号？？？

- master进程管理子进程的数据结构

  - ngx_processes全局数组

    ```c
    // 最多只能有1024个子进程
    #define NGX_MAX_PROCESSES 1024
    
    // 当前进程在ngx_processes数组中的下标
    ngx_int_t ngx_process_slot;
    // ngx_processes数组中有意义的元素的最大下标
    ngx_int_t ngx_last_process;
    // 存储所有子进程的数组
    ngx_process_t ngx_processes[NGX_MAX_PROCESSES];
    ```

  - ngx_process_t结构体

    ```c
    typedef struct {
        // ...
    } ngx_process_t;
    ```

  - 启动子进程

    - 启动子进程的方法

      ```c
      ngx_pid_t ngx_spawn_process(ngx_cycle_t *cycle, 
                                  ngx_spawn_proc_pt proc, 
                                  void *data, 
                                  char *name, 
                                  ngx_int_t respawn)
      ```

    - 函数指针ngx_spawn_proc_pt

      - 指针形式

        ```c
        typedef void (*ngx_spawn_proc_pt) (ngx_cycle_t *cycle, void *data);
        ```

      - ngx_spawn_process的proc参数就是子进程要执行的工作循环

        - worker进程的工作循环

          ```c
          static void ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
          ```

        - cache manage进程或者cache loader进程的工作循环

          ```c
          static void ngx_cache_manager_process_cycle(ngx_cycle_t *cycle, void *data)
          ```

  - 改变ngx_processes数组的元素状态
    - 依旧通过信号
    - 图8-8
      - ###



