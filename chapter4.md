# 第4章 配置、error日志和请求上下文

- 4.1回顾第2章中http配置项的特点
- 4.2全面讨论如何使用http配置项
  - 使用Nginx预设的解析方法
  - 自定义配置项的解析方式
- 4.3从HTTP框架的实现机制上解释http配置项的模型
- 4.4讨论Nginx为用户准备好的输出日志方法
- 4.5探讨维持一个请求的必要信息的上下文
  - 上下文与全异步实现的Nginx服务之间的关系
  - 如何使用HTTP上下文
  - HTTP框架如何管理请求的上下文结构体

## 4.1 http配置项的使用场景

- 一个例子

  ```json
  http {
  	test_str main;
      
      server {
      	listen 80;
      	test_str server80;
      
      	location /url1 {
      		mytest;
     			test_str loc1;
  		}
  
  		location /url2 {
              mytest;
              test_str loc2;
          }
  	}
  
  	server {
          listen 8080;
          test_str server8080;
          location /url3 {
          	mytest;
          	test_str loc3;
      	}
      }
  }
  ```

- 这个例子里，mytest模块会处理两个监听端口（8080、80）上建立的TCP连接，以及3种HTTP请求（请求URL分别对应/url1、/url2、/url3）

- 假设mytest要取出test_str配置项的参数，但是test_str出现了不同的参数值，那么渠道的test_str值以哪个为准？

  - 在每一个http块、server块或location块下都会生成独立的数据结构在存放配置项
  - 当请求是/url1时，test_str的值应当是location块下的loc1，还是这个location所属于的server块下的server80，还是其所属的http块下的值main，这个完全由mytest模块自己决定（用户自定义这个行为，见4.2）

## 4.2 怎样使用http配置

- 第3章中已经使用过mytest配置项
  - mytest配置项没有值，只是用来标识当location块内出现mytest配置项时就启用mytest模块
- HTTP模块如何获得感兴趣的配置项
  - 处理http配置项分为以下步骤
    - 创建数据结构用于存储配置项
    - 设定配置项在nginx.conf中出现时的限制条件与回调方法
    - 实现第2步中的回调方法，或使用Nginx框架预设的14个回调方法
    - 合并不同级别的配置快中出现的同名配置项
  - 以上步骤和Nginx结合
    - 通过第3章介绍的两个数据结构ngx_http_module_t和ngx_command_t

### 4.2.1 分配用于保存配置参数的数据结构

- 自定义一个结构体来存储感兴趣的配置项的参数

  ```c
  typedef struct {
      ngx_str_t   	my_str;
      ngx_int_t   	my_num;
      ngx_flag_t   	my_flag;
      size_t				my_size;
      ngx_array_t*  my_str_array;
      ngx_array_t*  my_keyval;
      off_t   			my_off;
      ngx_msec_t   	my_msec;
      time_t   			my_sec;
      ngx_bufs_t   	my_bufs;
      ngx_uint_t   	my_enum_seq;
      ngx_uint_t		my_bitmask;
      ngx_uint_t   	my_access;
      ngx_path_t*		my_path;
  
      ngx_str_t			my_config_str;
      ngx_int_t			my_config_num;
  } ngx_http_mytest_conf_t;
  ```

  - 每一个配置块都会有自己独立的ngx_http_mytest_conf_t结构（4.1）

  - 如何管理用户自定义的ngx_http_mytest_conf_t结构体？

    - 通过ngx_http_module_t的回调方法（第3章）

    - 回忆ngx_http_module_t的结构

      ```c
      // 只列出来和管理ngx_http_mytest_conf_t结构体相关的方法
      typedef struct {
        	// ...
        	char *(*create_main_conf)(ngx_conf_t *cf);
          char *(*create_srv_conf)(ngx_conf_t *cf);
        	char *(*create_loc_conf)(ngx_conf_t *cf);
        	// ...
      } ngx_http_module_t;
      ```

    - 解析nginx.conf过程中，遇到http{...}配置块时，HTTP框架会调用所有HTTP模块的create_main_conf、create_srv_conf、create_loc_conf方法（如果有实现的话），生成存储main级别配置参数的结构体

      - 因为每个HTTP模块都有可能对这个http配置块感兴趣，所以会调用所有HTTP模块的这三个方法
      - 为什么要调用三个方法，而不是只调用create_main_conf方法？？？
      - 









## 4.3 HTTP配置模型

- 本节套理论HTTP配置模型如何实现，第10章从HTTP框架的角度讨论它怎么管理每一个HTTP模块的配置结构体

- 在http{...}块中，一个ngx_http_conf_ctx_t结构保存了所有HTTP模块的配置数据结构的入口（也就是说，ngx_http_conf_ctx_t可以找到存放配置项参数的所有结构体）

- ngx_http_conf_ctx_t结构

  ```c
  typedef struct {
      // 数组中的每个元素指向HTTP模块的create_main_conf方法产生的结构体
      void **main_conf;
      void **srv_conf;
      void **loc_conf;
  } ngx_http_conf_ctx_t;
  ```

- 每当遇到http{...}块的时候，通过1个ngx_http_conf_ctx_t结构保存了所有HTTP模块的配置数据结构的入口

  - 遇到server{...}和location{...}也一样

### 4.3.1 解析HTTP配置的流程

- 解析流程见图4-1
  1. Nginx进程的主循环代码，这部分代码循环解析配置项，为什么要循环？？？
  2. 发现配置文件中含有http{}时，HTTP框架开始启动（10.7的ngx_http_block）
  3. 
     - 初始化每一个HTTP模块，设定它们的序列号
     - 创建3个数组，用于存储所有HTTP模块的create_xxx_conf方法返回的地址指针（也就是它们创建出来的用于保存配置项的数据结构），并把这3个数组的地址保存到ngx_http_conf_ctx_t中
  4. 调用每个HTTP模块的create_xxx_conf方法（如果实现）
  5. 返回create_xxx_conf返回的地址，并把它们依次保存到3个数组
  6. 调用每个HTTP模块的preconfiguration方法（如果实现）？？？
  7. preconfiguration若返回失败，Nginx进程会停止
  8. HTTP框架开始循环解析nginx.conf文件中http{...}里的所有配置项，直到第19步才返回
  9. 配置文件解析器在检测到一个配置项后，会遍历所有的HTTP模块，检查它们的ngx_command_t数组中的name项是否与配置项名相同
  10. 如果找到有一个HTTP模块（如mytest模块）对这个配置项感兴趣，就调用ngx_command_t结构的set方法来处理（见4.2？？？）
  11. set方法返回是否成功，失败则会停止Nginx进程
  12. 配置文件解析器继续监测配置项，如果发现server{...}配置项，就调用ngx_http_core_module模块来处理（规定好的）
  13. 在解析server{...}之前，也会像第3步那样，只不过这次和main级别相关的那个数组为空，然后像第4步那样调用每个HTTP模块的create_xxx_conf方法（没有main）
  14. 参考第5步，把create_xxx_conf返回的指针（保存配置项的数据结构）保存到ngx_http_conf_ctx_t的数组里
  15. 参考第8步，开始调用配置文件解析器来处理server{...}里的配置项
  16. 参考第9步，遍历当前server{...}的所有配置项
  17. 当前server块已经到尾部，说明server块内的配置项处理完毕，返回ngx_http_core_module
  18. http core模块处理完server配置项，返回配置文件解析器，解析后面的配置项
  19. 配置文件解析器处理到http{...}的尾部返回给HTTP框架继续处理
  20. 第3步（发现http{...}）和第13步（发现server{...}）以及其它一些步骤都创建了ngx_http_conf_ctx_t结构，这时会开始调用merge_srv_conf、merge_loc_conf等方法合并这些块中，每个HTTP模块分配的数据结构（4.2.5）
  21. HTTP框架处理完http配置项（ngx_command_t结构中的set回调方法），返回给配置文件解析器继续处理http{...}外的配置项
  22. 配置文件解析器处理完所有配置项，告诉Nginx主循环，配置项解析完毕，Nginx启动服务器

### 4.3.2 HTTP配置模型的内存布局

#### 4.3.2.1 http块和server块下存储配置项参数的结构体之间的关系

- 图4-2
  - ngx_http_conf_ctx_t有三个元素
  - main_conf指向一个数组，这个数组的元素是HTTP模块的create_main_conf产生的指针
  - 数组的指针元素指向存储配置项参数的结构体
  - server块下的ngx_http_conf_ctx_t结构中的main_conf会直接指向http块下的ngx_http_conf_ctx_t的main_conf（server块中没有main级别的配置）

#### 4.3.2.2 server块和location块下存储配置项参数的结构体之间的关系

- 图4-3

#### 4.3.2.3 http{}、server{}、location{}块的个数和create_xxx_conf调用次数的关系

- 一旦解析到http{}块，会调用所有HTTP模块的create_xxx_conf方法创建三组结构体，来存放各个HTTP模块感兴趣的main级别配置项
- 一旦解析到server{}块，会调用所有HTTP模块的create_srv_conf、create_loc_conf方法创建两组结构体，来存放各个HTTP模块感兴趣的srv级别配置项
- 一旦解析到location{}块...
- 为什么需要这么多结构体来存放配置项？
  - 为了解决同名配置项的合并问题

### 4.3.3 如何合并配置项

#### 4.3.3.1 为什么要合并配置项

- 一个例子

  ```json
  http {
  	listen 80;
      server {
      	listen 8080;
  	}
  	
  	server {
          // 没有listen参数
      }
  }
  ```

- 在这个例子中，某个模块ngx_http_mytest_module希望这样使用配置：第一个server的监听端口为8080，第二个server的监听端口用http块的80
- 这个时候就需要通过合并配置项，使得有listen配置项的第一个server使用自己的监听端口，没有listen配置项的第二个server使用所在http块的监听端口

#### 4.3.3.2 相关数据结构

![image-20210309143348763](/home/xiaogouguohe/.config/Typora/typora-user-images/image-20210309143348763.png)

#### 4.3.3.3 合并配置项的流程

- 图4-4
- 代码解释见10.2
- 需要合并的数据结构见4.3.3.2

### 4.3.4 预设配置项处理方法的工作原理

###