PHP函数
============

###创建第一个PHP扩展函数

在PHP扩展中，创建一个函数主要需要经过三步：

1. 在源文件(.c)中使用`PHP_FUNCTION`宏创建函数实现，并头文件中声明该函数
2. 使用`PHP_FE`告诉`zend_function_entry`结构体新创建的函数的地址
3. 将`zend_function_entry`结构体注册到`zend_module_entry`扩展入口结构体上，只有
创建第一个函数的时候需要这样做。

接下来，我们对这三个步骤展开，并且辅以一个名为`demo_array()`的函数作为例子，该函数返回一个
我们在扩展函数中创建的数组作为返回值。

在讲解如何创建一个扩展函数之前，我们需要创建一个扩展的基本骨架，创建扩展的基本骨架请参考
[PHP扩展开发 - 构建第一个PHP扩展]。

在[PHP扩展开发 - 构建第一个PHP扩展]中，我们创建了一个名为`ext_demo_1`的扩展程序，进入扩展目录，
我们将看到如下文件：


    /vagrant/ext/ext_demo_1$ ls
    config.m4   CREDITS       ext_demo_1.c    php_ext_demo_1.h
    config.w32  EXPERIMENTAL  ext_demo_1.php  tests


首先，我们需要在`ext_demo_1.c`文件中，创建函数的实现：


    PHP_FUNCTION(demo_array)
    {
        zval *subarray;/* 子数组 */

        array_init(return_value); /* 将函数返回值初始化为数组类型 */

      /* 返回数组中添加三个值：life=>42, 123=>1, 124=>3.1415926 */
        add_assoc_long(return_value, "life", 42);
        add_index_bool(return_value, 123, 1);
        add_next_index_double(return_value, 3.1415926);

      /* 添加两个字符串值： 125=> Foo, 126=> Bar */
        add_next_index_string(return_value, "Foo", 1);
        add_next_index_string(return_value, estrdup("Bar"), 0);

      /* 初始化zval结构体，分配内存空间 */
        MAKE_STD_ZVAL(subarray);
        array_init(subarray);

        add_next_index_long(subarray, 1);
        add_next_index_long(subarray, 20);
        add_next_index_long(subarray, 132);

      /* 将subarray添加到返回值 */
        add_index_zval(return_value, 444, subarray);
    }


创建函数体之后，我们需要在头文件`php_ext_demo_1.h`中声明该函数。


	PHP_FUNCTION(demo_array);


第二步是告诉`zend_function_entry`结构体函数的地址。在`ext_demo_1.c`文件的第 **41** 行左右，
我们可以看到`zend_function_entry`结构体变量，将函数通过`PHP_FE`宏添加到该变量数组中。


    const zend_function_entry ext_demo_1_functions[] = {
        PHP_FE(confirm_ext_demo_1_compiled,	NULL)		/* For testing, remove later. */
        PHP_FE(demo_array, NULL) /* 在这里添加demo_array函数 */
        PHP_FE_END	/* Must be the last line in ext_demo_1_functions[] */
    };


一般来说，如果使用的是`ext_skel`创建的扩展骨架的话，一个函数就算是添加完成了，因为第三步在生成扩展骨架的时候已经自动的完成了，
这里的第三步就是将该`ext_demo_1_functions`添加到`zend_module_entry`结构体上。


    zend_module_entry ext_demo_1_module_entry = {
    #if ZEND_MODULE_API_NO >= 20010901
        STANDARD_MODULE_HEADER,
    #endif
        "ext_demo_1",
        ext_demo_1_functions,  /* 注意这里，添加了 ext_demo_1_functions 变量 */
        PHP_MINIT(ext_demo_1),
        PHP_MSHUTDOWN(ext_demo_1),
        PHP_RINIT(ext_demo_1),		/* Replace with NULL if there's nothing to do at request start */
        PHP_RSHUTDOWN(ext_demo_1),	/* Replace with NULL if there's nothing to do at request end */
        PHP_MINFO(ext_demo_1),
    #if ZEND_MODULE_API_NO >= 20010901
        "0.1", /* Replace with version number for your extension */
    #endif
        STANDARD_MODULE_PROPERTIES
    };

为了验证我们的函数创建是否成功，我们编译一下这个扩展:


    # phpize
    # ./configure
    # make
    # make install


> 注意： 如果之前编译过该扩展，需要先`make clean`一下，清理掉上次编译产生的中间文件。

安装完成之后，验证一下是否成功：


    $ php -r "print_r(demo_array());"
    Array
    (
        [life] => 42
        [123] => 1
        [124] => 3.1415926
        [125] => Foo
        [126] => Bar
        [444] => Array
            (
                [0] => 1
                [1] => 20
                [2] => 132
            )

    )


好了，一个函数就已经创建完成了，在php文件中，我们就可以直接调用刚才创建的函数了：

    <?php
    print_r(demo_array());


###函数结构解析

为了对该函数的创建过程有个直观的了解，我们对刚才用到的宏进行简单的剖析。

这里的`PHP_FUNCTION`实际上是Zend定义的一个宏，展开后如下：


    #define PHP_FUNCTION(name) \
        void zif_##name(INTERNAL_FUNCTION_PARAMETERS)


也就是说，如果有函数定义如下：


    PHP_FUNCTION(sample_hello_world)
    {
      php_printf("Hello World!\n");
    }

在编译的时候将会被替换为：


    void zif_sample_hello_world(
      int ht,
      zval *return_value,         /* 函数返回值 */
      zval **return_value_ptr,
      zval *this_ptr,
      char return_value_used TSRMLS_DC     /* 标识返回值是否被使用了 */
    ){
      ...
    }


> 这里的 **zif_** 是*Zend Internal Functions* 缩写，为了避免定义的函数与C内部函数名冲突。


对于函数`demo_array`，内部实现如下：


    ZEND_FUNCTION(demo_array)
    {
      ...
    }


部分宏替换之后如下：

    void zif_demo_array(int ht, zval *return_value, zval **return_value_ptr, zval *this_ptr, int return_value_used TSRMLS_DC)
    {
      ...
    }

当然，这样还没有结束，Zend引擎并不知道该函数的地址，因此，需要告诉引擎函数地址：

    const zend_function_entry ext_demo_1_functions[] = {
        PHP_FE(confirm_ext_demo_1_compiled,	NULL)		/* For testing, remove later. */
        PHP_FE(demo_array, NULL)
      PHP_FALIAS(demo_array_alias, demo_array, NULL) /* 函数别名 */
        PHP_FE_END	/* Must be the last line in ext_demo_1_functions[] */
    };

> 注意，zend_function_entry结构体最后一组值为`PHP_FE_END` ({ NULL, NULL, NULL, 0, 0 })，如果需要添加新的函数，则
> 在上面添加PHP_FE宏即可。

> 这里的`PHP_FALIAS`宏为函数demo_array提供了一个别名demo_array_alias。

> 使用zif前缀仍然可能与内部函数名称产生冲突，可以使用`PHP_NAMED_FUNCTION`和`PHP_NAMED_FE`
> 配合使用（与PHP_FUNCTION和PHP_FE一样）

这里的`PHP_FE`定义如下：


    #define PHP_FE			ZEND_FE   /* php.h:341 */

    #define ZEND_FE(name, arg_info)						ZEND_FENTRY(name, ZEND_FN(name), arg_info, 0)  /* zend_API.h:77 */

    #define ZEND_FENTRY(zend_name, name, arg_info, flags)	{ #zend_name, name, arg_info, (zend_uint) (sizeof(arg_info)/sizeof(struct _zend_arg_info)-1), flags },


该宏需要两个参数，第一个参数为函数名，第二个为参数，用于提供参数提示信息。


###如何使用函数返回值

最直接的方法返回值是通过下面的方式：


    PHP_FUNCTION(sample_long)
    {
      RETVAL_LONG(return_value, 42); /* 本质上是ZVAL_LONG，可读性考虑 */
      return;
    }

> 这里的`return_value`为PHP_FUNCTION宏展开后，zif_sample_long函数的参数。

RETVAL_系列宏包含: `RETVAL_BOOL`, `RETVAL_NULL`, `RETVAL_TRUE`, `RETVAL_FALSE`,
`RETVAL_LONG`, `RETVAL_DOUBLE`, `RETVAL_STRING`, `RETVAL_STRINGL`, `RETVAL_RESOURCE` (zend_API.h:584)。

通常情况下，函数返回值之后就会退出当前函数，因此，通常会使用`RETURN_*`系列函数，与上面的`RETVAL_*`
系列类似，具体查看源码 zend_API.h 第596行左右。


###关于`zend_parse_parameters()`

zend_parse_parameters()的第一个参数为`ZEND_NUM_ARGS() TSRMLS_CC`，该参数返回函数
参数的个数，第二个参数为是一个字符串，包含了每个参数的类型标识符，具体类型标识符及其所代表的含义
见下表，其它参数根据提供的类型标识符不同而不同。

类型标识符  | 用户空间数据类型                             | C数据类型
---------|------------------------------------------ |---------
b        | Boolean                                   | zend_bool
l        | Integer                                   | long
d        | Float point                               | double
s        | String                                    | char*, int
r        | Resource                                  | zval*
a        | Array                                     | zval*
o        | Object instance                           | zval*
O        | Object Instance of a specified type       | zval\*, zend_class_entry\*
z        | None-specific zval                        | zval*
Z        | Dereferenced non-specific zval            | zval\*\*

下面的代码段展示了`zend_parse_parameters()`函数的使用方法：


    PHP_FUNCTION(demo_parameter)
    {
        long age = 24;
        char *name;
        int name_len;

        if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "s|l", &name, &name_len, &age) == FAILURE) {
            RETURN_NULL();
        }


        PHPWRITE(name, name_len);
        php_printf(" 您好，您的年龄是 %ld\n", age);
        RETURN_TRUE;
    }

> 注意的是，对于类型s和类型O，对应的参数为两个。`s` 为字符串类型，提供两个参数（变量内容，长度），
> `O`为指定类型的对象实例（对象zval，对象类型）

下表是`zend_parse_parameters()`支持的类型修饰符：

类型修饰符        | 含义
---------------|--------
 &brvbar;      | 在它之前的参数是必选的，之后的是可选的
 !             | !修饰之前的类型标识符，表明该参数如果手动传值为NULL的话，会将该变量的指针设为NULL指针，而不是创建一个NULL结构体变量
 /             | /修饰之前的类型标识符，表明该参数会被指定为复制时写，在创建该变量的时候，会将is_ref=0,refcount=1。

> 如果没有/，变量会按照写时复制（更新时复制）的方式传递，将ref\_count_\_gc=2, is\_ref\__gc=1，
> 这样，如果需要修改变量值的话，需要进行变量分离，比较麻烦，可以指定/标识符，这样，在进入函数的时候，
> 就已经是分离的变量了：ref\_count_\_gc=1, is\_ref\_\_gc=0。

例如:

    /* 这里如果设置参数为NULL，将会出啊功能键一个zval类型的val变量，设置其值为NULL，
      这样会占用一定的CPU时钟周期，虽然为NULL，但是也占用资源。*/
    PHP_FUNCTION(sample_arg_fullnull)
    {
      zval *val;
      if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z",
                                      &val) == FAILURE) {
          RETURN_NULL();
      }
      if (Z_TYPE_P(val) == IS_NULL) {
          val = php_sample_make_defaultval(TSRMLS_C);
      }
    ...
    /* 这里使用了参数标识符z!，说明该参数如果设置为NULL的话，将不会创建zval变量，而是将其
     指针设置为空指针 */
    PHP_FUNCTION(sample_arg_nullok)
    {
      zval *val;
      if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z!",
                                      &val) == FAILURE) {
          RETURN_NULL();
      }
      if (!val) {
          val = php_sample_make_defaultval(TSRMLS_C);
      }
    ...



###参数类型提示和校验

参数类型提示功能是在Zend Engine 2之后加入的，因此，对于PHP4来说，该功能不可用。要使用参数类型提示，
需要在`ZEND_BEGIN_ARG_INFO()`或者`ZEND_BEGIN_ARG_INFO_EX()`宏和`ZEND_END_ARG_INFO()`之间，添加`ZEND_ARG_*INFO()`系列宏(zend_API.h: 101-110)。


    /* 注意：这里仅仅列出了宏名称和参数，为避免篇幅过长，实现部分已经删除，请查看源码zend_API.h 101-110 */
    #define ZEND_ARG_INFO(pass_by_ref, name)
    #define ZEND_ARG_PASS_INFO(pass_by_ref)
    #define ZEND_ARG_OBJ_INFO(pass_by_ref, name, classname, allow_null)
    #define ZEND_ARG_ARRAY_INFO(pass_by_ref, name, allow_null)
    #define ZEND_BEGIN_ARG_INFO_EX(name, pass_rest_by_reference, return_reference, required_num_args)
    #define ZEND_BEGIN_ARG_INFO(name, pass_rest_by_reference)
    #define ZEND_END_ARG_INFO()


从上述代码可以看出，对于`ZEND_BEGIN_ARG_INFO_EX()`宏，可以接受四个参数：

- **name** 该参数是函数名称标识，比如定义函数`demo_array`，则此处可以为`demo_array_args`。
- **pass_rest_by_reference**  函数参数是否为引用传递，如果为0为否，1为是。
- **return_reference** 该参数是函数返回值是否是以引用返回，0为值返回，1为引用返回。
- **required_num_args** 必须参数个数（也可以说是前几个参数必须），如果为-1则为所有参数都必须

`ZEND_BEGIN_ARG_INFO`宏与`ZEND_BEGIN_ARG_INFO_EX()`宏一样，只不过是对后者进行了包装，
只需要提供前两个参数即可，返回值为按照值返回（非引用），所有参数必须。
这里的`ZEND_RETURN_VALUE`宏在`zend_compile.h`文件中第244-245行左右：


    #define ZEND_RETURN_VALUE				0
    #define ZEND_RETURN_REFERENCE			1


可以看到，`ZEND_ARG_*INFO`系列宏一共有四个，涉及到四个参数：

- **pass_by_ref** 该值为是否按照引用传递
- **name** 参数名称
- **classname** 参数的类名
- **allow_null** 是否允许为NULL值

下面是`PHP Yaf` 框架中`yaf_controller.c`文件中对控制器的`render`方法进行类型提示的一小段代码：


    ZEND_BEGIN_ARG_INFO_EX(yaf_controller_render_arginfo, 0, 0, 1)
        ZEND_ARG_INFO(0, tpl)
        ZEND_ARG_ARRAY_INFO(0, parameters, 1)
    ZEND_END_ARG_INFO()


可以看出，`render`函数接收两个参数，并且这两个参数都是按照值传递，返回值也是按照值传递的方式，
只有第一个`tpl`参数是必须参数，`parameters`参数为可选参数，并且该参数为数组，并且允许为NULL值。

对于函数`demo_parameter`，我们可以创建如下的类型提示代码：


    ZEND_BEGIN_ARG_INFO_EX(demo_parameter_args, 0, 0, 1)
        ZEND_ARG_INFO(0, name)
        ZEND_ARG_INFO(0, age)
    ZEND_END_ARG_INFO()

添加上述代码之后，我们需要告诉Zend引擎该参数提示信息是提供给`demo_parameter`函数的，
在`zend_function_entry`变量`ext_demo_1_functions`中，修改`PHP_FE(demo_parameter,NULL):`:


	PHP_FE(demo_parameter, demo_parameter_args)


编译扩展，测试是否有效：


    ext_demo_1 mylxsw$ php -r "echo demo_parameter();"
    PHP Warning:  demo_parameter() expects at least 1 parameter, 0 given in Command line code on line 1
    PHP Stack trace:
    PHP   1. {main}() Command line code:0
    PHP   2. demo_parameter() Command line code:1

    Warning: demo_parameter() expects at least 1 parameter, 0 given in Command line code on line 1

    Call Stack:
        0.0001     219264   1. {main}() Command line code:0
        0.0001     219312   2. demo_parameter() Command line code:1



[PHP扩展开发 - 构建第一个PHP扩展]: http://aicode.cc/article/397.html
