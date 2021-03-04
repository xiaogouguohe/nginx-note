# 第7章 nginx提供的高级数据结构

## 7.1 Nginx提供的高级数据结构概述

### 7.1.1 ngx_queue_t双向链表

- 与Nginx的内存池无关
  - 也就是说，这个链表不负责分配内存来存放链表元素，只是把这些已经分配好内存的元素用双向链表连接起来
- 只有两个指针的额外空间
- 提供插入排序法

### 7.1.2 ngx_array_t动态数组

- 类似STL的vector容器
- 自动扩容

### 7.1.3 ngx_list_t单项链表

- 负责内存的分配，和ngx_queue_t不同
- 见3.2.3

### 7.1.4 ngx_rbtree_t红黑树

### 7.1.5 ngx_radix_tree_t基数树

- 和红黑树一样都是二叉查找树
- 以整型数据作为关键字，应用范围比红黑树小
- 插入、删除元素不需要旋转，因此效率比红黑树高

### 7.1.6 支持通配符的散列表

- Nginx首先实现了基础的常用散列表
- 在此基础上，对于URI域名的场景设计了支持通配符的散列表
  - 只支持前置通配符和后置通配符

## 7.2 ngx_queue_t双向链表

### 7.2.1 为什么设计ngx_queue_t双向链表

- ngx_queue_t的优势

  - 有顺序容器的优势
  - 相对于Nginx其它顺序容器的优势
    - 实现排序功能
    - 不负责内存的分配，额外开销小
    - 支持两个链表间的合并

- nginx_queue_t的实现

  ```c
  typedef struct ngx_queue_s ngx_queue_t;
  
  struct ngx_queue_s {
      ngx_queue_t *prev;
      ngx_queue_t *next;
  }
  ```

### 7.2.2 双向链表的使用方法

- 双向队列容器和队列中的元素用的都是同一个数据结构ngx_queue_t
- Nginx分别封装了对容器和元素操作的方法，见表7-1和表7-2

### 7.2.3 使用双向链表排序的例子

- 这个例子中可以看到
  - 如何定义、初始化ngx_queue_t容器
  - 如何自定义任意类型的链表元素
  - 如何遍历链表
  - 如何自定义排序方法进行排序

- 定义链表元素的结构体

  ```c
  typedef struct {
      u_char* str;
      ngx_queue_t qEle;
      int num;
  } TestNode;
  ```

  - 链表元结构体中包含ngx_queue_t类型的成员

- 排序方法需要自定义，现在以TestNode结构体中的num成员作为排序依据

  ```c
  ngx_int_t compTestNode(const ngx_queue_t* a, const ngx_queue_t* b) {
      TestNode* aNode = ngx_queue_data(a, TestNode, qEle);
      TestNode* bNode = ngx_queue_data(b, TestNode, qEle);
      return aNode->num > bNode->num;
  }
  ```

  - ngx_queue_data的定义

    ```c
    /* offsetof返回link成员在type结构体中的偏移量（4.2.2）
        ngx_queue_t类型的指针，向前偏移qEle对于TestNode的偏移量，
        也就是回到TestNode的起始地址，
        这样不会踩到别的成员变量（栈溢出）？？？*/
    #define ngx_queue_data(q,type,link) \
    (type *) ((u_char *) q - offsetof(type, link))
    ```

- 定义双向链表容器queueContainer，并初始化为空链表

  ```c
  ngx_queue_t queueContainer;
  ngx_queue_init(&queueContainer);
  ```

- 定义若干个TestNode结构体作为链表元素，并且初始化num成员

  ```c
  int i = 0;
  testNode node[5];
  for (; i < 5; i++) {
      node[i].num = i;
  }
  ```

- 把这些TestNode结构体添加到queueContainer链表中
  
- 见书上代码
  
- 遍历
  
- 见书上代码
  
- 排序

  ```c
  ngx_queue_sort(&queueContainerqueueContainerqueueContainer, compTestNode);
  ```

### 7.2.4 双向链表是如何实现的

- 见图7-2~图7-4

## 7.3 ngx_array_t动态数组

### 7.3.1 为什么设计ngx_array_t动态数组

- 访问数组很快
- 分配的数组太大容易浪费空间，因此用动态数组
- 分配内存由内存池管理

### 7.3.2 动态数组的使用方法

- 动态数组的定义

  ```c
  typedef struct ngx_array_s ngx_array_t;
  
  struct ngx_array_s {
      /* 数组的首地址 */
      void *elts;
      /* 数组中已经使用的元素个数 */
      ngx_uint_t nelts;
      /* 每个数组元素占用的内存大小 */
      size_t size;
      /* 当前数组的容量 */
      ngx_uint_t nalloc;
      /* 内存池对象 */
      ngx_pool_t *pool;
  };
  ```

- 动态数组的内存结构见图7-5

- 动态数组的方法见表7-3

### 7.3.3 使用动态数组的例子

- 以7.2.3中介绍的TestNode作为数组中的元素类型

  ```c
  /* 预分配了1个元素的空间，
      每个元素占用的内存字节数为sizeof(TestNode) */
  ngx_array_t* dynamicArray = ngx_array_create(cf->pool, 1, sizeof(TestNode));
  
  /* 返回新添加元素的地址 */
  TestNode*a = ngx_array_push(dynamicArray);
  a->num = 1;
  /* 发生扩容 */
  a = ngx_array_push(dynamicArray);
  a->num = 2;
  
  /* 一次性添加3个元素,
      返回的是新添加的第一个元素的地址*/
  TestNode* b = ngx_array_push_n(dynamicArray, 3);
  b->num = 3;
  (b + 1)->num = 4;
  (b + 2)->num = 5;
  
  /* 遍历数组，从数组的首地址开始 */
  TestNode* nodeArray = dynamicArray->elts;
  ngx_uint_t arraySeq = 0;
  for (; arraySeq < dynamicArray->nelts; arraySeq++) {
      /* 遍历到元素a */
      a = nodeArray + arraySeq;
      /* do something */
  }
  
  /* 销毁已分配的数组元素空间和ngx_array_t对象 */
  gx_array_destroy(dynamicArray);
  ```

### 7.3.4 动态数组的扩容方式

- 每次扩容的大小受制于内存池的以下两种情形
  - 如果当前内存池剩余的空间不小于本次要新增的空间，那么本次扩容只扩容新增的空间
  - 否则
    - 对ngx_array_push方法来说，会把原来动态数组的容量扩容一倍
    - 对于ngx_array_push_n方法来说
      - 如果n小于原来动态数组的容量，会扩容一倍
      - 否则，会扩大至2*n
    - 需要把原动态数组的元素复制到新的动态数组中

## 7.4 ngx_list_t单向链表

###

## 7.5 ngx_rbtree_t红黑树

- Nginx的核心模块（定时器、文件缓存模块等）在需要快速检索的情况下用了ngx_rbtree_t容器
- 本节使用一个例子：有10个元素要存储到红黑树中，分别为1、6、8、11、13、15、17、22、25、27

### 7.5.1 为什么设计ngx_rbtree_t红黑树

- 防止二叉树太深

### 7.5.2 红黑树的特性

###

### 7.5.3 红黑树的使用方法

- 红黑树节点ngx_rbtree_node_t

  ```c
  typedef ngx_uint_t ngx_rbtree_key_t;
  typedef struct ngx_rbtree_node_s ngx_rbtree_node_t;
  
  struct ngx_rbtree_node_s {
      ngx_rbtree_key_t key;
      ngx_rbtree_node_t *left;
      ngx_rbtree_node_t *right;
      ngx_rbtree_node_t *parent;
      /* 0表示黑，1表示红 */
      u_char color;
      /* 1个字节的节点数据，
          空间太小，很少使用，
          红黑树对数据的管理类似于双向链表，
          链表本身不保存数据，而是节点作为成员贯穿在数据中 */
      u_char data;
  }
  ```

- 把ngx_rbtree_node_t“贯穿”到结构体，一般作为第一个成员，方便把结构体强制转类型成ngx_rbtree_node_t

  ```c
  typedef struct {
      ngx_rbtree_node_t node;
      ngx_uint_t num;
  } TestRBTreeNode;
  ```

- 红黑树结构体ngx_rbtree_t

  ```c
  typedef struct ngx_rbtree_s ngx_rbtree_t;
  
  /* 对于关键字冲突的情况，
      用户可以自定义冲突的行为（替换或者新增等），
      表7-4提供了3种实现 */
  typedef void (*ngx_rbree_insert_pt) (ngx_rbtree_node_t *root, ngx_rbtree_node_t *node, ngx_rbtree_node_t *sentinel);
  
  struct ngx_rbtree_s {
      ngx_rbtree_node_t *root;
      /* 哨兵节点NIL */
      ngx_rbtree_node_t *sentinel;
      ngx_rbtree_insert_pt insert;
  }
  ```

- 节点ngx_str_node_t

  ```c
  /* 用于关键字为字符串的情况 */
  typedef struct {
      ngx_rbtree_node_t node;
      ngx_str_t str;
  } ngx_str_node_t;
  ```

- 红黑树的方法
  - 表7-5
- 红黑树节点的方法
  - 表7-6

### 7.5.4 使用红黑树的例子

###

### 7.5.5 如何自定义添加成员方法

###



