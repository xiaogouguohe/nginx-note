# 第9章 事件模块

- ngx_event_t事件和ngx_connection_t连接是处理TCP连接的基本数据结构
- 在此基础上，9.4探讨核心模块ngx_events_module
  - 它定义了一种模块类型——事件模块
- 9.5说明第1个事件模块ngx_event_core_module
  - 它的职责是如何管理当前正在使用的事件驱动模式，如Nginx启动时决定到底是基于select还是epoll
- 9.6
  - 深入LInux内核，研究epoll的实现原理和使用方法
  - ngx_epoll_module模块
- Nginx的定时器事件见9.7
  - 基于第7章的红黑树
- 9.8综合性地介绍事件处理框架
  - ngx_process_events_and_timers方法处理网络事件
  - 定时器事件
  - post事件
  - 如何解决“惊群”
  - 如何均衡worker的连接数

- 9.9介绍异步I/O如何实现高效读取磁盘

## 9.1 事件处理框架概述

- 事件处理框架要解决的问题是如何收集、管理、分发事件（8.2.2.2）

- 事件主要以网络事件和定时器事件为主

  - 定时器事件见9.7

- Nginx如何收集、管理TCP网络事件

  - 网络事件的驱动和不同的操作系统以及内核版本有关

  - 因此为每个操作系统提供的事件驱动都是不同的

    - Linux内核2.6之前的版本或大部分类Unix操作系统都可以使用poll（ngx_poll_module模块实现）或者select（ngx_select_module模块实现）
    - Linux内核2.6之后的版本可以使用epoll（ngx_epoll_module模块实现）
    - FreeBSD上可以使用kqueue（ngx_kqueue_module模块实现）
    - ...

  - 如何选定合适的事件驱动机制

    - 核心模块ngx_events_module

      - Nginx启动时会调用ngx_init_cycle方法解析配置项，一旦找到ngx_events_module感兴趣的"events{}"配置项，ngx_events_module就会开始工作

      - ngx_events_module的工作

        - 定义了事件类型的模块
        - 为所有事件模块解析events中的配置项，调用其在ngx_command_t数组中定义的回调方法

        - 管理这些事件模块的配置结构体

    - ngx_event_core_module

      - 决定使用哪种事件驱动机制
      - 如何管理事件
      - 9.5会详细讨论ngx_event_core_module模块在启动过程中的工作
      - 9.8会在时间框架的运行中再见到ngx_event_core_module

    - 运行在不同操作系统、不同内核版本的事件驱动模块

  - 事件驱动模块接口

    - 什么是“模块的接口”

      - ngx_module_t表示Nginx模块的接口

      - 针对于每种类型的模块，都有一个结构体来描述这一类模块的通用接口

      - 这个接口保存在ngx_module_t结构体的ctx成员中

      - 核心模块的通用接口是ngx_core_module_t结构体（8.2.1）

        ```c
        typedef struct {
            ngx_str_t name;
            void *(*create_conf)(ngx_cycle_t *cycle);
            char *(*init_conf)(ngx_cycle_t *cycle, void *conf);
        } ngx_core_module_t;
        ```

      - 事件模块的通用接口是ngx_event_module_t结构体

        ```c
        typedef struct {
            ngx_str_t *name;
            // 解析配置项前，创建存储配置项参数的结构体
            void *(*create_conf)(ngx_cycle_t *cycle);
            // 解析配置项后，处理当前事件模块感兴趣的全部配置项
            char *(*init_conf)(ngx_cycle_t *cycle, void *conf);
            
            // 没个事件模块需要实现的10个抽象方法
            ngx_event_actions_t actions;
        } ngx_event_module_t;
        ```

      - actions的10个抽象方法

        ```c
        typedef struct {
            // ...
            // 把一个感兴趣的事件添加到事件驱动机制（如epoll、kqueue等）中，
            // 这样，在事件发生后，可以在调用下面的process_events时获取这个事件
            ngx_int_t (*add)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
        } ngx_event_actions_t;
        ```

      - ngx_event_core_module和9个事件驱动模块都必须实现ngx_event_module_t接口
        - 9.5介绍ngx_event_core_module如何实现
        - 9.6介绍ngx_epoll_module如何实现

## 9.2 Nginx事件的定义

### 9.2.1 事件结构体

- 事件结构体

  ```c
  typedef struct ngx_event_s ngx_event_t;
  struct ngx_event_s {
      // ...
      // 消费事件的方法，每个事件模块都会实现它
      ngx_event_hadler_pt handler;
      // ...
  };
  ```

- handler方法的原型

  ```c
  typedef void (*ngx_event_handler_pt)(ngx_event_t *ev);
  ```

- 后续会见到很多handler方法的实现

### 9.2.2 操作事件的方法

#### 9.2.2.1 ngx_cycle_t结构体相关成员

```c
typedef struct ngx_cycle_s ngx_cycle_t;
struct ngx_cycle_s {
    // ...
    // 当前进程中所有连接对象的总数
    ngx_uint_t connection_n;
    // 当前进程中所有连接对象
    ngx_connection_t *connections;
    // 当前进程中的所有读事件对象，
    // connection_n同时表示所有读事件的总数
    // 读、写事件到底有哪些？？？
    ngx_event_t *read_events;
    // 当前进程中的所有写事件对象
    // connection_n同时表示所有写事件的总数
    ngx_event_t *write_events;
    // ...
}
```

#### 9.2.2.2 事件不需要创建

- Nginx在启动一个进程时已经在ngx_cycle_t的read_events成员中预分配所有读事件，在write_events成员中预分配所有写事件
- 图9-1中，每个连接自动对应一个读事件和一个写事件

#### 9.2.2.3 把事件添加到epoll等事件驱动模块中

- ngx_event_actions_t结构体的add和del方法可以实现在事件驱动模块中添加或移除事件

- Nginx封装一层

  - 添加读事件

    ```c
    // 读事件添加到事件驱动模块中
    // 这个方法是属于具体某个模块的，否则如何添加到这个模块中？？？
    // 该事件对应的TCP连接上一旦出现可读事件，就会回调该事件的handler方法
    // rev是要添加的事件
    // flags指定事件的驱动方式（一般可以忽略）
    ngx_int_t ngx_handle_read_event(ngx_event_t *rev, ngx_uint_t flags);
    ```

  - 添加写事件

    ```c
    // lowat表示当连接对应的套接字缓冲区中不许有lowat大小的剩余空间，
    // 时间收集器（如select或epoll_wait）才会处理这个可写事件，
    //（llowat为0时不考虑可写缓冲区的大小）
    ngx_int_t ngx_handle_write_event(ngx_event_t *wev, size_t lowat);
    ```

## 9.3 Nginx连接的定义

- 每个用户请求至少对应一个TCP连接
- 为了及时处理连接，至少需要一个读事件和一个写事件，使得epoll可以根据触发的事件调度相应的模块读取请求或发送响应
  - 保留两个事件结构体，不用每次需要的时候都再申请，可以快速给到事件分发器

- 两种连接
  - 被动连接（9.3.1）
    - 客户端主动发起、Nginx服务器被动接受的TCP连接
    - 数据结构ngx_conection_t表示这种连接
  - 主动连接（9.3.2）
    - Nginx向上游服务器建立连接
    - 数据结构ngx_peer_connection_t结构表示这种连接
- 连接不可以随意创建，要从连接池中获取（9.3.3）

### 9.3.1 被动连接

###

### 9.3.2 主动连接

###

### 9.3.3 ngx_connection_t连接池

- ngx_connection_t结构体相关成员

  ```c
  typedef struct ngx_connection_s ngx_connection_t;
  struct ngx_connection_s {
      /*连接未使用时，data成员充当next指针，指向下一个空闲连接；
      连接被使用时，data的意义由使用它的Nginx模块而定，
      如在HTTP框架中，data指向ngx_http_request_t请求？？？*/
      void *data;
      
      /* 连接对应的读事件 */
      ngx_event_t *read;
      /* 连接对应的写事件 */
      ngx_event_t *write;
      // ...
  }
  ```

- ngx_cycle_t相关成员

  ```c
  typedef struct ngx_cycle_s ngx_cycle_t;
  struct ngx_cycle_s {
      // ...
      
      ngx_uint_t free_connection_n;
      ngx_connection_t *free_connections;
      
      // 当前进程中所有连接对象的总数
      ngx_uint_t connection_n;
      // 当前进程中所有连接对象
      ngx_connection_t *connections;
      // 当前进程中的所有读事件对象，
      // connection_n同时表示所有读事件的总数
      ngx_event_t *read_events;
      // 当前进程中的所有写事件对象
      // connection_n同时表示所有写事件的总数
      ngx_event_t *write_events;
      // ...
  }
  ```

- 图9-1
  - connections指向整个连接池数组
  - free_connections指向第一个ngx_connnection_t空闲连接
  - ngx_connection_t的data指向下一个空闲连接
  - 事件池（和连接池一一对应）

- 操作连接池的方法

  - 获取连接

    ```c
    /* s是这条连接的套接字句柄
    log是记录日志的对象 */
    ngx_connection_t *ngx_get_connection(ngx_socket_t s, ngx_log_t *log);
    ```

  - 回收连接

    ```c
    void ngx_free_connection(ngx_connection_t *c);
    ```

## 9.4 ngx_events_module核心模块

- ngx_events_module的功能

  - 9.1

- ngx_events_module的定义

  ```c
  ngx_module_t ngx_events_module = {
      NGX_MODULE_V1,
      &ngx_events_module_ctx,		
      ngx_events_commands,
      NGX_CORE_MODULE,
      NULL,
      NULL,
      NULL,
      NULL,
      NULL,
  	NULL,
      NULL,
      NGX_MODULE_V1_PADDING
  };
  ```

- ngx_events_module的ngx_command_t成员

  ```c
  static ngx_command_t ngx_events_commands[] = {
      {
          ngx_string("events"), // 数组只有一个有意义的成员，说明ngx_events_module这个核心模块只对块配置项events{...}感兴趣
          NGX_MAIN_CONF | NGX_CONF_BLOCK | NGX_CONF_NOARGS,
          ngx_events_block, // 处理块配置项events{...}的方法
          0,
          0,
          NULL
      },
  
      ngx_null_command
  }
  ```

  - ngx_events_module只对一个块配置项感兴趣，就是events{...}配置项
  - 如何处理events{...}配置项见第10章

- 核心模块共同接口ngx_core_module_t

  ```c
  static ngx_core_module_t ngx_events_module_ctx = {
      ngx_string("events"),
      NULL, /* create_conf */
      NULL /* init_conf */
  };
  ```

  - 只定义了模块名字，没有实现create_conf和init_conf方法
    - ngx_events_module不会解析配置项的参数，而是在出现events配置项后调用各事件模块解析events{...}的配置项，核心模块如何“唤醒”非核心模块开始配置项解析？？？
    - 因此不需要create_conf方法创建存储配置项参数的结构体，也不需要init_conf方法处理解析的配置项

### 9.4.1 如何管理所有事件模块的配置项

- 内存结构见图9-2
  - 每个事件模块通过接口ngx_event_module_t的create_conf方法，来创建配置结构体指针
  - 配置结构体指针被放到ngx_events_module核心模块创建的指针数组中，这个指针数组就是图9-2中的“所有事件模块的配置结构体指针”
  - 注意和图4-2比较
  - ngx_http_module模块会不会和图4-2有联系？？？
- 每一个事件模块如何获取它在create_conf中分配的结构体指针
  - ###

### 9.4.2 管理事件模块

- 配置结构体指针的保存在ngx_events_block方法中进行，方法的执行步骤见图9-3

  1. 初始化所有事件模块的ctx_index成员

     - ngx_module_t的相关成员

       ```c
       struct ngx_module_s {
       	// ...
           ngx_uint_t ctx_index;
           ngx_uint_t index;
           // ...
       }
       ```

     - 在启动Nginx后，调用ngx_init_cycle方法（第8章）前初始化index

       ```c
       ngx_max_module = 0;
       for (i = 0; ngx_modules[i]; i++) {
           ngx_modules[i]->index = ngx_max_movule++;
       }
       ```
       - 各模块在ngx_modules数组中的顺序是不能乱的

     - 同一个类型的模块间的顺序通过ctx_index来维护

       - index效率低

     - ngx_events_block方法初始化ctx_index的代码

       ```c
       ngx_event_max_module = 0;
       for (i = 0; ngx_modules[i]; i++) {
           if (ngx_modules[i]0>type != NGX_EVENT_MODULE) {
               continue;
           }
           
           ngx_modules[i]->ctx_index = ngx_event_max_module++;
       }
       ```

  2. 分配9.4.1中的指针数组
  3. 依次调用事件模块通用接口ngx_event_module_t的create_conf方法，产生的配置结构体指针放在第2步的指针数组中
  4. 对所有事件模块解析配置项（根据每个事件模块的ngx_command_t来解析）
  5. 依次调用事件模块通用接口ngx_event_module_t的init_conf方法

## 9.5 ngx_event_core_module事件模块

- ngx_event_core_module是事件类型的模块

  - 在所有事件模块中的顺序是第一位，先于其它事件模块执行
  - 要做的任务
    - 创建连接池和读写事件（9.3）
    - 决定使用的事件驱动机制
    - 初始化要使用的事件模块

- ngx_event_core_module感兴趣的配置项

  - 通过ngx_command_t数组来获取感兴趣的配置项，代码见书上

  - 对配置项参数的解析使用了Nginx预设的配置项解析方法（第4章）
  - 该模块的配置结构体ngx_event_conf_t见书上
    - 与负载均衡锁相关的两个成员（9.8）

- ngx_event_core_module实现ngx_event_module_t接口

  - 每个事件模块都要实现ngx_event_module_t接口
  - ngx_event_core_module只实现了create_conf和init_conf方法，没实现ngx_event_actions_t方法，因为它不负责TCP网络事件的驱动
  - 实现代码见书上

- ngx_event_core_module定义见书上

  - 实现了ngx_event_module_init和ngx_event_process_init方法

  - Nginx启动过程中，还没有fork子进程时，会首先调用ngx_event_module_init方法

    - 初始化一些变量

  - 在fork出worker子进程后，每个worker进程在调用ngx_event_process_init方法后才会进入正式的工作循环

  - ngx_event_process_init方法的流程见图9-4

    1. 打开负载均衡锁，

       - 需要满足以下条件
         - 配置文件打开负载均衡锁
         - master模式
         - worker进程数大于1
       - 3个有关变量

    2. 若不满足第1步中的3个条件，则关闭负载均衡锁

    3. 初始化红黑树实现的定时器（9.6）

    4. 在调用use配置项指定的事件模块中，调用ngx_event_module_t接口的ngx_event_actions_t成员的init方法进行这个事件模块的初始化

    5. 如果配置文件设置了timer_resolution配置项，说明要控制时间精度

       - 调用setitimer方法，设置时间间隔为timer_resolution毫秒来回调ngx_timer_signal_handler方法

       - ngx_tier_signal_handler方法

         ```c
         void ngx_tier_signal_handler(int signo) {
             ngx_event_timer_alarm = 1;
         }
         ```

       - 在ngx_event_actions_t的process_events方法中，每个事件驱动模块都要在ngx_timer_alarm为 1时调用ngx_timer_update方法（9.7.1）更新系统时间，更新后把ngxx_event_timer_alarm设为0

    6. 若使用了epoll，则会为ngx_cycle_t结构体重的files成员预分配句柄

    7. 预分配连接池

    8. 预分配读事件池

    9. 预分配写事件池

    10. 把读/写事件和连接对象一一对应，并且串联成链表（图9.1）

    11. 处理ngx_cycle_t的free_connections

    12. 为所有ngx_listening_t监听对象中的connection成员分配连接，有新连接事件时会调用ngx_event_accept方法建立新连接（9.8）
    13. 将监听对象连接的读事件（？？？）添加到事件驱动模块中，这样epoll等事件模块就开始检测监听服务，向用户提供服务了

## 9.6 epoll事件驱动模块

- 本节会讨论Linux是如何实现epoll事件驱动机制的
- 在此基础上，会进一步说明ngx_epoll_module模块是如何基于epoll实现Nginx的事件驱动的

### 9.6.1 epoll的原理和用法

- select和poll

  - 如何实现的，见《Unix环境高级编程》第14章相关代码（用select实现一个服务端-多客户端模型）
  - 有什么缺点
    - 把所有连接的套接字都传给操作系统内核，实际上有事件发生的连接可能只有很少

- epoll的实现

  - 在内核中申请了一个简易的文件系统，把原先的select调用分成了三个部分

    - epoll_create建立一个epoll对象
    - epoll_ctl向epoll对象添加所有连接的套接字，新连接添加，断开连接则删除
    - epoll_wait收集发生事件的连接

  - epoll_create方法创建event_poll结构体

    ```c
    struct eventpoll {
        /* 红黑树根节点，树存储着所有添加到epoll的事件（epoll_ctl）
        红黑树使得重复添加的事件很容易被识别出来*/
        struct rb_root rbr;
        /* 双向链表，保存着将要通过epoll_wait返回给用户的事件 */
    	struct list_head rdllist;
    };
    ```
    
    - 每个epoll对象都有一个eventpoll结构体，它会在内核空间创造内存

  - 对于每个事件都会建立一个epitem结构体

    ```c
    struct epitem {
        // ...
        /* 红黑树节点，贯穿 */
        struct rb_node rbn;
        /* 双向链表节点，贯穿 */
        struct list_head rdllink;
        /* 事件句柄 */
        struct epoll_filefd ffd;
        /* 所属的eventpoll对象 */
        struct eventpoll *ep;
        /* 期待的事件类型（这个连接发生了什么事，比如可以读、写，连接发生错误等等） */
        struct epoll_event event;
        // ...
    };
    ```

    - epoll_event

      ```c
      struct epoll_event {  
          __uint32_t events;   
          epoll_data_t data;
      };
      ```

      - events的取值见表9-3，表明这个连接发生了什么

      - data

        ```c
        typedef union epoll_data {
            void *ptr;
            int fd;
            uint32_t u32;
            uint64_t u64;
        } epoll_data_t;
        ```

        ###

  - epoll_wait检查是否有发生事件的连接

    - 只检查eventpoll对象中的rdllist双向链表是否有epitem元素，如果有就把这里的事件复制到用户态内存，同时把事件数量返回给用户

### 9.6.2 如何使用epoll

- epoll通过3个epoll系统调用为用户提供服务

  1. epoll_create

     ```c
     /* 返回一个epoll的句柄，
         size是大致的连接数目（在新内核版本中size没有意义），
         不再使用epoll时记得close句柄 */
     int epoll_create(int size);
     ```

  2. epoll_ctl

     ```c
     int epoll_ctl(int epfd, int op, int fd, struct epoll_veent*event);
     ```

     - 向epoll对象中添加、修改或者删除感兴趣的事件
     - 返回0表示成功，否则返回小于0，根据错误码判断错误类型
     - epfd是epoll_create返回的句柄，表示一个epoll对象
     - op见表9-2，表示增加、删除或修改
     - fd是事件的句柄
     - event告诉epoll对什么事件感兴趣
       - 见9.6.1的epoll_event

  3. epoll_wait系统调用

     ```c
     int epoll_wait(int epfd, struct epoll_event* events, int max_events int timeout);
     ```

     - 收集epfd这个epoll监控的事件中已经发生的事件
     - 如果没有任何事件发生，最多等待timeout毫秒后返回

     - 返回值表示发生的事件个数，0表示没有事件发生，-1表示错误
     - 发生的事件会复制到events数组中，数组需要用户自己申请空间
     - maxevents表示可以返回的最大事件数目，通常和events大小相等