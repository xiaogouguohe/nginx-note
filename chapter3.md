# 第3章 开发一个简单的HTTP模块

- 如何开发一个HTTP模块
  - 把程序嵌入到Nginx（3.3）
  - 这个HTTP模块要能介入到HTTP请求的处理流程（3.1、3.4、3.5）
  - 一些Nginx框架定义的数据结构（3.2）
  - 正式处理请求时，还要可以获得Nginx框架接收、解析后的用户请求信息（3.6）
  - 发送响应给用户（3.7）
  - 将瓷盘中的文件以HTTP包体的形式发送给用户（3.8）
- 如何用C++编写HTTP模块（3.9）

## 3.1 如何调用HTTP模块

###

## 3.2 准备工作

- 本节介绍Nginx定义的几个基本数据结构和方法
- 第7章还会介绍一些复杂的容器

### 3.2.1 整型的封装

```c
typedef intptr_t ngx_int_t; /* 有符号整型 */
typedef uintptr_t ngx_uint_t; /* 无符号整型 */
```

### 3.2.2 ngx_str_t数据结构

```c
typedef struct {
    size_t len; /* 字符串的有效长度 */
    u_char *data; /* 字符串起始地址 */
} ngx_str_t;
```

### 3.2.3 ngx_list_t数据结构

- 定义

  ```c
  typedef struct ngx_list_part_s ngx_list_part_t;
  
  // 链表的一个元素（节点）
  struct ngx_list_part_s {
      void *elts;
      ngx_uint_t nelts; /* 当前数组已使用的元素个数 */
      ngx_list_part_t *next; 
  };
  
  /*整个链表，
      链表的每个节点ngx_list_part_t存储一个数组，
      数组的容量为size * nalloc */
  typedef struct {
      ngx_list_part_t *last;
      ngx_list_part_t part;
      size_t size;
      ngx_uint_t nalloc;
      ngx_pool_t *pool;
  } ngx_list_t;
  ```

  - ngx_list_t是一个链表，链表的节点是一个数组，数组的每个元素最大大小为size，数组容量nalloc

- 这样设计的好处

  - 要存储的数据是灵活的，可以是任意类型的数据结构，只要大小不超过size
  - 占用的内存有ngx_list_t管理
  - 小块内存通过链表访问效率低下，通过数组访问更高效

- ngx_list_t的成员pool是管理内存分配的内存池对象

- ngx_list_t的内存分布情况见图3-2

  - 一般情况下，内存尽量是连续的

- 相关接口

  - ngx_list_create创建新链表

    ```c
    /* pool是内存池对象，
        size是节点中每个元素的大小（相当于ngx_list_t->size），
        n是每个链表节点可容纳的元素的个数（相当于ngx_list_t->nalloc）*/
    ngx_list_t *ngx_list_create(ngx_pool_t *pool, ngx_uint_t n, size_t size);
    ```

  - ngx_list_init使用方法与ngx_list_create类似

  - ngx_list_push添加新的元素

    ```c
    ngx_list_t* testlist = ngx_list_create(r->pool, 4, sizeof(ngx_str_t));
    if (testlist == NULL) {
        retrun NGX_ERROR;
    }
    
    /* 返回新分配的节点的首地址 */
    ngx_str_t *str = ngx_list_push(testlist);
    if (str == NULL) {
        return NGX_ERROR;
    }
    
    str->len = sizeof("Hello world");
    str->value = "Hello world";
    ```

  - 遍历链表
    
    - 见书上代码

### 3.2.4 ngx_table_elt_t数据结构

- 定义

  ```c
  /* key/value对，为HTTP头部量身定制 */
  typedef struct {
      ngx_uint_t hash; /* 用于快速检索头部（3.6.3） */
      ngx_str_t key;
      ngx_str_t value;
      u_char *lowcase_key; /* key的全小写 */
  } ngx_table_elt_t;
  ```

### 3.2.5 ngx_buf_t数据结构

###

### 3.2.6 ngx_chain_t数据结构

###

## 3.3 如何将自己的HTTP模块编译进Nginx

###

## 3.4 HTTP模块的数据结构

### 3.4.1 Nginx模块的数据结构

- ngx_module_t结构体

  ```c
  typedef struct ngx_module_s ngx_module_t;
  struct ngx_module_s {
      // ...
  };
  ```

### 3.4.2 HTTP模块

- 定义HTTP模块时，要把ngx_module_t的type字段设为NGX_HTTP_MODULE



### 3.4.3 commands数组

- ngx_module_t的成员，每个元素是ngx_command_t类型

- ngx_command_t结构

  ```c
  typedef struct ngx_command_s ngx_command_t;
  struct ngx_command_s {
      ngx_str_t 	name;
      ngx_unit_t 	type;
      char *(*set)(ngx_conf_t *Cf, ngx_command_t *cmd, void *conf);
      ngx_uint_t 	conf;
      ngx_uint_t 	offset;
      void*				post;
  }
  ```

- ngx_command_t作用

  - 每个模块有若干个ngx_command_t，每个ngx_command_t表示这个模块可能会用到（感兴趣）的配置项
  - nginx.conf的配置项有很多，每个模块都会从所有配置项中选择自己感兴趣的配置项，如何筛选？？？
  - 筛选完这个模块感兴趣的配置项以后，需要进行解析，见4.2



