# nginx配置文件的解析（1）

这次主要想探讨一下，nginx的配置文件nginx.conf，是怎么被解析成有意义的结构，又是怎么被nginx的各个模块使用的。除了必要的一些数据结构以外，这次会尽量避开代码层面的讨论（因为其实我不懂），而且大部分内容也来自《深入理解nginx第2版》。

## 1 nginx.conf结构

首先，需要简单看一下nginx.conf的结构。

```
http {
    #...
    server {
				#...
				location / {
						#...
				}
				location /test {
						mytest;
				}
    }
}
```

## 2 模块结构ngx_module_t

还要了解一下nginx用来表示模块的数据结构，比较详细的内容可以去查《深入理解nginx第2版》3.4的内容，这里只列出来现在要用到的一些成员。

```c
typedef struct ngx_module_s ngx_module_t;
struct ngx_module_s {
  	// ... 
  	void 					*ctx;
  	ngx_command_t *commands;
  	// ...
}
```

## 3 ngx_command_t

当ngx_module_t表示一个http模块的时候，ctx会指向一个ngx_module_http_t结构（会在第4节提到），还有一个成员是ngx_command_t数组：

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

### 3.1 ngx_command_t的作用：筛选配置项

ngx_command_t的作用是，定义了感兴趣的一个配置项。这句话可以这样去理解，配置文件里有很多配置项，nginx也有很多模块，包括官方提供的和我们自定义的。但是对于一个模块而言，它并不一定会用到所有的配置项，而是要进行筛选。那么，怎么筛选出自己要用到的配置项呢？从ngx_module_t的定义中也可以看到，每个模块ngx_module_t结构中会有一系列的ngx_command_t，每个ngx_command_t都会去nginx.conf中筛选出一些配置项（应该和ngx_command_t的name和type成员有关，具体筛选规则可以先不用太关心）。这样，每个模块结构都可以筛选出自己要用到的配置项。

### 3.2 如何解析配置项：自定义结构体

筛选出来这些配置项之后，怎么去解析呢？因为配置项只是一串文本，具体有什么意义，还要看解析的规则。但是不管解析规则是什么，首先还是要把文本解析成struct这种结构。nginx对于配置项的解析可能不是很好理解，因为它是把配置项解析到用户自定义的struct的。也就是说，用户根据业务需要，自己定义一个struct来存放这个配置项解析出来的结果。听起来有点奇怪，因为如果nginx没有统一的某个struct类型来管理解析出来的配置项，那么对于不同用户定义出来的不同struct，似乎根本无法管理。但是不管怎么说，我们先按照这个事实，继续我们的讨论，等到后面就可以逐步厘清nginx是怎么管理这些不同的struct类型了。

现在，我们先自己定义一个结构体，来存放解析出来的配置项：

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

这个结构体，就是用来保存某一条配置项的参数的。配置项的参数指的是配置项后面紧跟的内容，比如说之前展示的配置文件的第一个location是一个配置项，它的参数就是 / 和 大括号这一整段文本，也就是说location的参数是两个；当然也有些配置项没有参数的，比如mytest就是没有参数的配置项。

### 3.3 ngx_command_t的偏移量成员

现在问题来了，ngx_http_mytest_conf_t 只保存了某个配置项的参数，我们怎么知道这些参数是哪个配置项的参数呢？因为配置项有很多条，那么ngx_http_mytest_conf_t也会有很多个，怎么样找到某条配置项的参数是放在哪个ngx_http_mytest_conf_t呢？找到这个ngx_http_mytest_conf_t之后，又是在哪个成员呢？

想要解决这个问题，就要回去看ngx_command_s这个结构。conf和offset两个成员都是偏移量，想象一下内存当中有一系列排列好的ngx_http_mytest_conf_t数组，那么offset就可以定位到某一个ngx_http_mytest_conf_t，而offset就可以定位到这个ngx_http_mytest_conf_t的某个成员。回忆前面的内容，每个ngx_command_s都会根据成员name筛选出一条配置项（事实上可能不止一条，但是在这里我们先考虑只筛选出一条的情况），这条配置项的参数就是ngx_http_mytest_conf_t的这个成员。

### 3.4 nginx_command_t的函数指针set成员

顺着这个思路想下去，很容易就会引出来又一个问题，某个配置项的参数，应该怎么在ngx_http_mytest_conf_t的成员里表示呢？比如说，某个配置项的test_flag的参数是on，而根据偏移量计算出来这个参数应该保存在ngx_http_mytest_conf_t的成员my_flag，但是我们发现my_flag的类型是ngx_flag_t，是个long int类型。怎么办？我们总不能把"on"这个字符串赋给ngx_flag_t类型的my_flag，所以我们得另想办法。再回去看ngx_command_s这个结构，有个函数指针set。在c和cpp中，看到函数指针，就要意识到，这个地方可能是可以让用户自己实现这个类型的函数的。这里也的确是这样，当然nginx也提供了14个这种类型的函数（见《深入理解nginx第2版》118页）。它们的作用是，根据刚才提到的配置项参数test_flag的值，来写my_flag的值。比如说如果test_flag的值是on，就可以把my_flag设为1。

### 3.5 一个例子

我们用一个很简单的例子来总结一下3.1到3.4的内容。

``` c
//《深入理解nginx第2版》122页
static ngx_command_t ngx_http_mytest_commands[] = {{
  			// ...
  			ngx_string("test_flag"), // 筛选出配置名为"test_flag"的配置项
  			NGX_HTTP_LOC_CONF | NGX_CONF_FLAG, // 该配置项可以在哪些块出现（在哪些块出现才会被筛选）《深入理解nginx第2版》116页）
				ngx_conf_set_flag_slot, // 根据test_flag的值设my_flag的值（《深入理解nginx第2版》118页）
  			NGX_HTTP_LOC_CONF_OFFSET, // ngx_http_mytest_conf_t的位置
  			offsetof(ngx_http_mytest_conf_t, my_flag), // my_flag的位置
  			NULL
		},
    ngx_null_command
}
```

这个例子，ngx_http_my_test_command_t会根据前两个成员，在配置文件中筛选出来配置项，然后根据第四个成员，找到某个ngx_http_mytest_conf_t，再根据第五个成员，找到ngx_http_mytest_conf_t的my_flag成员，最后根据第三个成员和配置项test_flag的值，来确定my_flag的值。

这里还需要声明一点，一个ngx_http_mytest_conf_t是可以保存多个配置项的参数的，事实上，是ngx_http_mytest_conf_t和nginx.conf的配置块具有一一对应的关系，而不是ngx_http_mytest_conf_t和nginx.conf的配置项。这一点会在之后第4节提及。

## 4 何时构造ngx_http_mytest_conf_t：ngx_http_module_t

现在，我们搞清楚了ngx_command_t和ngx_http_mytest_conf_t的作用，ngx_command_t是为了从配置文件中筛选出配置项，ngx_http_mytest_conf_t是为了保存配置项的参数。那么现在就又有一个问题，nginx什么时候会构造一个ngx_http_mytest_conf_t结构体？回忆一下之前提到的ngx_http_module_t，这个表示http模块，在这里我们暂时需要关注这些成员：

```c
typedef struct {
  	// ...
  	char *(*create_main_conf)(ngx_conf_t *cf);
		char *(*create_srv_conf)(ngx_conf_t *cf);
  	char *(*create_loc_conf)(ngx_conf_t *cf);
  	// ...
} ngx_http_module_t;
```

这些都是创建ngx_http_mytest_conf_t的函数指针。又是函数指针。其实这也很好理解，因为之前已经说过，ngx_http_mytest_conf_t这个用来存放配置项参数的结构体是用户自定义的，别的用户可能会定义出另外一个结构体来存放配置项参数，因此创建struct的方法当然要通过函数指针实现。

那为什么会有三个函数指针呢？这个和配置文件的结构和解析流程有关。回忆一下之前提到的结构，nginx有很多个ngx_module_s结构，其中有些的类型是http模块，那么ngx_module_s的成员ctx就会指向一个ngx_http_module_t结构。因此，当解析http {...} 时，会依次执行所有模块的这三个方法（如果有实现的话）；而到了解析server {...} 时，只会执行所有模块的后两个方法；到解析location {...} 的时候，只会解析所有模块的最后一个方法。也就是说，在解析nginx.conf的时候，会创建ngx_http_mytest_conf_t结构体，这就解决了我们之前关于什么时候会创建ngx_http_mytest_conf_t的问题。这样的话，就是每个http块会对应一个ngx_http_mytest_conf_t，每个server块会对应一个ngx_http_mytest_conf_t，每个location块会对应一个ngx_http_mytest_conf_t，这就是之前提到的配置块和ngx_http_mytest_conf_t的一一对应的关系。

搞清楚这些，现在就要自己实现创建ngx_http_mytest_conf_t了，可以参考《深入理解nginx》第2版，115页的代码，当然这个不是这次讨论的重点。

## 5 总结

回顾一下这次提到的内容。我们这次的思路大概是这样的。

- 配置文件的结构
- 每个模块要筛选自己用到的配置项，通过一系列ngx_command_t筛选
- 筛选到配置项之后，解析到自定义结构体ngx_http_mytest_conf_t
- 有很多个ngx_http_mytest_conf_t的实例，不知道某条配置项存在ngx_http_mytest_conf_t哪个实例的哪个成员，通过ngx_command_t的两个偏移量定位
- 配置项的参数是字符串，而ngx_http_mytest_conf_t的成员不一定是字符串，通过ngx_command_t的函数指针set进行参数值和成员值的约定，如true对应1
- ngx_http_mytest_conf_t是自定义的结构体，因此通过ngx_http_module_t结构体的函数指针进行管理，也就是创建ngx_http_mytest_conf_t的函数由用户自定义



尚未提及的东西：ngx_http_conf_ctx_t，三个成员，分别三个create产生的一系列自定义结构体

2020.12.15