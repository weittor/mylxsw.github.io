PHP面向对象实现
==============

###创建第一个PHP扩展类

本节将会通过实现一个简单的PHP扩展类，介绍在PHP扩展开发过程中如何实现面向对象。

在PHP扩展实现中，类的创建主要包含三步：

1. 创建一个全局的`zend_class_entry`变量，用于存储类的入口。
2. 创建一个`zend_function_entry`结构体数组，用于存储类中包含的方法。
3. 在扩展的`MINIT`方法中注册类。

下面将对这三个步骤进行展开描述，我们将会继续在[PHP扩展开发 - 构建第一个PHP扩展]一节中创建的
`ext_demo_1`扩展的基础之上进行开发，这里我们所写的所有代码都在`ext_demo_1.c`文件中。

####创建一个简单的空类

首先，我们创建一个名为`php_democlass_entry`的`zend_class_entry`结构体变量，
该结构体变量实际存储了我们创建的类的入口。


	zend_class_entry *php_democlass_entry;


> 这里的`php_democlass_entry`在扩展源文件中是一个全局变量，为了使其它扩展可以使用我们创建的类，
> 这个全局变量应该在头文件中导出。

接下来，我们创建`zend_function_entry`结构体数组，这个数组与函数定义时的数组是一样的。


    const zend_function_entry ext_demo_1_democlass_functions[] = {
        PHP_FE_END
    };


与函数注册不同的是，对于类的创建，我们需要在`MINIT`方法中注册该类。


    PHP_MINIT_FUNCTION(ext_demo_1)
    {
        /* 创建一个临时类入口变量 */
        zend_class_entry temp_ce;
        INIT_CLASS_ENTRY(temp_ce, "DemoClass", ext_demo_1_democlass_functions);

        php_democlass_entry = zend_register_internal_class(&temp_ce TSRMLS_CC);
        return SUCCESS;
    }


在`MINIT`函数中，首先创建了一个`temp_ce`变量用于存储临时的类入口，接下来使用`INIT_CLASS_ENTRY`
宏初始化该变量，之后使用`zend_register_internal_class()`将该类注册到Zend引擎，
该函数会返回一个最终的类入口，将其赋值给前面创建的全局变量。

这里的`INIT_CLASS_ENTRY`是一个宏定义:


    /* zend_API.h 162-163 */
    #define INIT_CLASS_ENTRY(class_container, class_name, functions) \
        INIT_OVERLOADED_CLASS_ENTRY(class_container, class_name, functions, NULL, NULL, NULL)

它接受三个参数，第一个参数为**类容器**，也就是我们创建的`zend_class_entry`变量，
第二个参数为我们要创建的对象**名称**，第三个参数为我们创建的类包含哪些**函数**。

在使用`INIT_CLASS_ENTRY`之后，都执行了哪些操作呢？跟进该宏定义的实现代码后可以发现，
在该宏的定义中，首先为结构体(zend_class_entry)变量`class_container`设置`name`属性，
然后对该结构体变量进行初始化(zend_API.h 176-204)。

现在，我们就有了一个空的类，编译该扩展之后，我们可以使用`php --rc DemoClass`测试一下：

    ext_demo_1 mylxsw$ php --rc DemoClass
    Class [ <internal:ext_demo_1> class DemoClass ] {

      - Constants [0] {
      }

      - Static properties [0] {
      }

      - Static methods [0] {
      }

      - Properties [0] {
      }

      - Methods [0] {
      }
    }


####为PHP类添加方法

接下来，我们为我们刚才创建的空类添加一个名为`sayHello`的方法。

在`ext_demo_1.c`文件中，创建一个`sayHello`的方法：


    PHP_METHOD(DemoClass, sayHello)
    {
        char *name, *str_hello;
        int name_len, str_hello_len;

        if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "s", &name, &name_len) == FAILURE) {
            RETURN_NULL();
        }

        str_hello_len = spprintf(&str_hello, 0, "Hello, %s, You are welcome!\n", name);
        RETURN_STRINGL(str_hello, str_hello_len, 0);
    }


这里我们创建为`DemoClass`类创建了一个名为`sayHello`的方法，该方法接收一个字符串类型的参数，
并且返回`Hello, 提供的参数, You are welcome!`。

不要忘记在头文件`php_ext_demo_1.h`文件中声明一下，否则编译的时候会因为`PHP_ME`宏中使用了该方法而报错。


	PHP_METHOD(DemoClass, sayHello);


在`ext_demo_1_democlass_functions`结构体数组中加入该方法：


    const zend_function_entry ext_demo_1_democlass_functions[] = {
        PHP_ME(DemoClass, sayHello, NULL, ZEND_ACC_PUBLIC) /* sayHello方法为PUBLIC可见性 */
        PHP_FE_END
    };


这里的`PHP_ME`宏与之前函数部分中`PHP_FE`类似，区别在于增加了第一个参数，用于指定该方法所属的类名，
最后一个参数用于指定方法属性。这里的方法属性包含
`ZEND_ACC_PUBLIC`，`ZEND_ACC_PROTECTED`，
`ZEND_ACC_PRIVATE`，`ZEND_ACC_STATIC`，
`ZEND_ACC_FINAL`，`ZEND_ACC_ABSTRACT`。

其中，前三个参数可以与后面几个组合使用，多个参数组合时，使用`|`进行分隔， 例如:


    PHP_ME(
        Test, protectedFinalStaticMethod, arginfo_xyz,
        ZEND_ACC_PROTECTED | ZEND_ACC_FINAL | ZEND_ACC_STATIC
    )


对于抽象方法来说，并不能直接使用`ZEND_ACC_ABSTRACT`宏，而应该使用Zend定义的
`PHP_ABSTRACT_ME(类名, 抽象方法名, 参数信息)`取代。

类似于`PHP_FUNCTION`宏，这里的`PHP_METHOD`宏展开后如下所示：

    /* PHP_METHOD(ClassName, methodName) { } */
    void zim_ClassName_methodName(INTERNAL_FUNCTION_PARAMETERS) { }


> 对于sayHello方法，我们可以使用ZEND_ARG_INFO系列宏为其参数提供类型提示功能。
>
    ZEND_BEGIN_ARG_INFO_EX(democlass_sayhello_args, 0, 0, 1)
        ZEND_ARG_INFO(0, name)
    ZEND_END_ARG_INFO();

> 添加该段代码之后，需要修改PHP_ME函数的第三个参数NULL为`democlass_sayhello_args`。

重新编译扩展，执行以下PHP脚本测试是否扩展功能正常：

    <?php
    $demo = new DemoClass();
    echo $demo->sayHello('mylxsw');

程序输出：

    $ php test.php
    Hello, mylxsw, You are welcome!


> 在类方法内，使用`getThis()`方法获取当前对象实例，返回值类型为`zval *`，对应PHP中的`$this`。

####为类添加属性

要在创建好的类中添加属性，要使用`zend_declare_property_*`系列函数。在`MINIT`方法中，
添加如下代码：

    PHP_MINIT_FUNCTION(ext_demo_1)
    {
        ...
        zend_declare_property_long(php_democlass_entry, "age", sizeof("age") - 1, 24, ZEND_ACC_PUBLIC TSRMLS_CC);
        ...
    }

这里我们使用`zend_declare_property_long()`函数为`DemoClass`类添加了一个`age`字段，
并且设置其可见性为PUBLIC，类型为long型。

下面是`zend_declare_property_*`系列函数:

    ZEND_API int zend_declare_property(zend_class_entry *ce, char *name, int name_length, zval *property, int access_type TSRMLS_DC);

    ZEND_API int zend_declare_property_ex(zend_class_entry *ce, const char *name, int name_length, zval *property, int access_type, char *doc_comment, int doc_comment_len TSRMLS_DC);

    ZEND_API int zend_declare_property_null(zend_class_entry *ce, char *name, int name_length, int access_type TSRMLS_DC);

    ZEND_API int zend_declare_property_bool(zend_class_entry *ce, char *name, int name_length, long value, int access_type TSRMLS_DC);

    ZEND_API int zend_declare_property_long(zend_class_entry *ce, char *name, int name_length, long value, int access_type TSRMLS_DC);

    ZEND_API int zend_declare_property_double(zend_class_entry *ce, char *name, int name_length, double value, int access_type TSRMLS_DC);

    ZEND_API int zend_declare_property_string(zend_class_entry *ce, char *name, int name_length, char *value, int access_type TSRMLS_DC);

    ZEND_API int zend_declare_property_stringl(zend_class_entry *ce, char *name, int name_length, char *value, int value_len, int access_type TSRMLS_DC);


要访问类中的属性，比如获取属性的值或者是修改属性的值等，使用`zend_read_property()`和
`zend_update_property()`函数。


    /* zend_API.h 326-328 */
    ZEND_API zval *zend_read_property(zend_class_entry *scope, zval *object, char *name, int name_length, zend_bool silent TSRMLS_DC);

    ZEND_API zval *zend_read_static_property(zend_class_entry *scope, char *name, int name_length, zend_bool silent TSRMLS_DC);

    ZEND_API void zend_update_property(zend_class_entry *scope, zval *object, char *name, int name_length, zval *value TSRMLS_DC);

    /* 下面这个是zend_update_property_*系列函数，与declare系列类似，不做详述，参考zend_API.h 309-328 */
    ZEND_API void zend_update_property_long(zend_class_entry *scope, zval *object, char *name, int name_length, long value TSRMLS_DC);


为了演示，我们注册一个名为`age`的long型属性，并创建对应的getter/setter方法。


    /* getAge方法的参数类型提示，无参数 */
    ZEND_BEGIN_ARG_INFO_EX(democlass_getage_args, 0, 0, 0)
    ZEND_END_ARG_INFO()

    /* setAge方法参数类型提示，一个必须参数，名称$age */
    ZEND_BEGIN_ARG_INFO_EX(democlass_setage_args, 0, 0, 1)
        ZEND_ARG_INFO(0, age)
    ZEND_END_ARG_INFO()

    /* 将创建的方法告诉zend_function_entry，设置可见性为public */
    const zend_function_entry ext_demo_1_democlass_functions[] = {
        ...
        PHP_ME(DemoClass, setAge, democlass_setage_args, ZEND_ACC_PUBLIC)
        PHP_ME(DemoClass, getAge, democlass_getage_args, ZEND_ACC_PUBLIC)
        PHP_FE_END
    };

    /* setAge($age)方法 */
    PHP_METHOD(DemoClass, setAge)
    {
        long age;
        zval *obj;
        if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "l", &age) == FAILURE) {
            RETURN_NULL();
        }
        obj = getThis();/* 获取$this对象 */
        /* 更新age字段的信息 */
        zend_update_property_long(php_democlass_entry, obj, "age", sizeof("age") - 1, age);
    }

    /* getAge方法*/
    PHP_METHOD(DemoClass, getAge)
    {
        zval *obj, *age;
        obj = getThis();
        /* 检索age属性的值 */
        age = zend_read_property(php_democlass_entry, obj, "age", sizeof("age") - 1, 1 TSRMLS_CC);
        RETURN_ZVAL(age, 1, 0);
    }

    /* MINIT方法*/
    PHP_MINIT_FUNCTION(ext_demo_1)
    {
        .../* 这里省略了，完成类的注册 */
        /* 声明属性age，并设置为public可见性 */
        zend_declare_property_long(php_democlass_entry, "age", sizeof("age") - 1, 24, ZEND_ACC_PUBLIC TSRMLS_CC);
        return SUCCESS;
    }


要创建static属性的话，同样使用`zend_declare_property*`系列函数，区别是类型标识符需要增加
`ZEND_ACC_STATIC`。对static字段的操作使用`zend_read_static_property()`和`zend_update_static_property*`系列函数。

还有一个是类常量，类常量使用`zend_declare_class_constant*`系列宏声明，使用`zend_update_class_constants`
进行操作。

####接口和继承

与在PHP中使用类和接口类似，在扩展开发中，扩展内部的类也可以继承其它类或者实现接口。

当我们创建的类需要继承一个已经存在的类的时候，我们只需要将注册类的函数修改为`zend_register_internal_class_ex`。


    designner_entry = zend_register_internal_class_ex(&temp_designner_ce, person_entry, NULL TSRMLS_CC);


与之前使用`zend_register_internal_class`不同的是，`_ex`版本的注册函数增加了两个参数，
第二个参数为要继承的类的全局标识，也就是在创建类的时候我们那个`zend_class_entry`全局对象。
这里第三个参数为NULL，这个参数的作用是在调用其它扩展类时，如果扩展没有按照规范导出类的全局标识符的话，
我们将第二个参数设置为NULL，第三个参数设为字符串形式的类名，当然，不推荐这样做，例如：


    custom_exception_ce = zend_register_internal_class_ex(
        &tmp_ce, NULL, "RuntimeException" TSRMLS_CC
    );


接口的创建与类相似，区别在于在接口创建时，在`zend_function_entry`中，需要将接口所有的方法
使用`PHP_ABSTRACT_ME`添加，其它步骤与类的创建一样，在MINIT方法中，当初始化类之后，只需要再
设置创建类的`ce_flags`字段为`ZEND_ACC_INTERFACE`即可。


    zend_class_entry *php_sample3_iface_entry;
    PHP_MINIT_FUNCTION(sample3)
    {
        zend_class_entry ce;
        INIT_CLASS_ENTRY(ce, "Sample3_Interface",
                            php_sample3_iface_methods);
        php_sample3_iface_entry =
                    zend_register_internal_class(&ce TSRMLS_CC);
        php_sample3_iface_entry->ce_flags|= ZEND_ACC_INTERFACE;/* 注意这里使用了“|” */
        ...


如果创建的类要实现接口，只需要再使用`zend_class_implements()`函数添加一下就可以了。


    /* DataClass implements Countable, ArrayAccess, IteratorAggregate */
    zend_class_implements(
            data_class_ce TSRMLS_CC, 3, spl_ce_Countable, zend_ce_arrayaccess, zend_ce_aggregate
        );


这里的`zend_class_implements()`函数是个变参函数，第一个参数为需要实现接口的类的zend_class_entry对象，第二个参数为需要实现的接口的个数，其它参数是可变的，都为需要实现的接口。

####对象创建

前面我们讲解了如何在PHP扩展开发中创建一个类，这里我们再说一说如何在扩展中实例化一个类，创建对象。
Zend引擎为创建类对象提供了以下三个宏：

    /* zend_API.h 359-351 下面宏定义已经展开 */
    int object_init(zval *arg);
    int object_init_ex(zval *arg, zend_class_entry *ce);
    int object_and_properties_init(zval *arg, zend_class_entry *ce, HashTable *properties);


实现范例：

    /* 创建一个SomeClass对象，并且传递初始化属性 */
    zval *obj;
    MAKE_STD_ZVAL(obj);
    object_and_properties_init(obj, class_entry_of_SomeClass, properties_hashtable);

    /* 创建一个SomeClass对象，使用默认属性 */
    zval *obj;
    MAKE_STD_ZVAL(obj);
    object_init_ex(obj, class_entry_of_SomeClass);
    /* = object_and_properties_init(obj, class_entry_of_SomeClass, NULL) */

    /* 创建一个stdClass对象 */
    zval *obj;
    MAKE_STD_ZVAL(obj);
    object_init(obj);
    /* = object_init_ex(obj, NULL) = object_and_properties_init(obj, NULL, NULL) */


当使用`object_init()`创建一个`stdClass`对象之后，我们可能需要为它添加一些属性，这时候需要注意的是，
不要使用`zend_update_property`函数，而是使用`add_property`宏。

    add_property_long(obj, "id", id);
    add_property_string(obj, "name", name, 1); // 1 means the string should be copied
    add_property_bool(obj, "isAdmin", is_admin);
    // also _null(), _double(), _stringl(), _resource() and _zval()




###附录


    typedef struct _zend_object {
        zend_class_entry *ce;
        HashTable *properties;
        HashTable *guards;
    } zend_object;


####对象结构

在`zval`中，与对象有关的字段是`zvalue_value`联合体中的`zend_object_value`字段，
该类型定义如下：


    /* zend_types.h 53-59 */
    typedef unsigned int zend_object_handle;
    typedef struct _zend_object_handlers zend_object_handlers;

    typedef struct _zend_object_value {
        zend_object_handle handle;
        zend_object_handlers *handlers;
    } zend_object_value;


该结构体`zend_object_value`中，`zend_object_handle`类型的handle为int类型的整数值，
该handle是一个唯一的对象ID标识，用于从对象存储中查询实际的对象。

第二个参数是一个指向`zend_object_handlers`结构体的指针，该结构体定义了实际对象的行为(zend_object_handlers.h 113-141)。



####类结构

在PHP扩展中，Zend引擎定义了`zend_class_entry`结构体来表示一个类的基本结构。

    /* zend.h 418-470 */
    struct _zend_class_entry {
        char type;/* 类的类型：ZEND_INTERNAL_CLASS 或者 ZEND_USER_CLASS */
        char *name;/* 类的名称 */
        zend_uint name_length;/* 类名称长度 ， sizeof(name) - 1 */
        struct _zend_class_entry *parent; /* 该类的父类 */
        int refcount;/* 引用计数 */
        zend_bool constants_updated;
      /*
       * ce_flags指定了类的基本属性
       *    ZEND_ACC_IMPLICIT_ABSTRACT_CLASS 隐式的抽象类（含有abstract方法）
       *    ZEND_ACC_EXPLICIT_ABSTRACT_CLASS 添加了abstract关键字的抽象类
       *    ZEND_ACC_FINAL_CLASS 该类是final的
       *    ZEND_ACC_INTERFACE 这是一个接口
       */
        zend_uint ce_flags;

        HashTable function_table; /* 方法哈希表 */
        HashTable default_properties; /* 默认属性哈希表 */
        HashTable properties_info; /* 属性信息哈希表 */
        HashTable default_static_members; /* 默认的静态成员变量哈希表 */
      /*
       * type == ZEND_USER_CLASS    ->  &default_static_members
       * type == ZEND_INTERAL_CLASS -> NULL
       */
        HashTable *static_members;
        HashTable constants_table; /* 常量哈希表 */
        const struct _zend_function_entry *builtin_functions; /* 方法定义入口 */

      /* 构造函数、析构函数、clone函数以及其它魔术方法 */
        union _zend_function *constructor;
        union _zend_function *destructor;
        union _zend_function *clone;
        union _zend_function *__get;
        union _zend_function *__set;
        union _zend_function *__unset;
        union _zend_function *__isset;
        union _zend_function *__call;
        union _zend_function *__callstatic;
        union _zend_function *__tostring;
        union _zend_function *serialize_func;
        union _zend_function *unserialize_func;

        zend_class_iterator_funcs iterator_funcs;

        /* 类句柄handler */
        zend_object_value (*create_object)(zend_class_entry *class_type TSRMLS_DC);
        zend_object_iterator *(*get_iterator)(zend_class_entry *ce, zval *object, int by_ref TSRMLS_DC);

      /* 类声明的接口 */
        int (*interface_gets_implemented)(zend_class_entry *iface, zend_class_entry *class_type TSRMLS_DC);
        union _zend_function *(*get_static_method)(zend_class_entry *ce, char* method, int method_len TSRMLS_DC);

        /* 序列化回调函数 */
        int (*serialize)(zval *object, unsigned char **buffer, zend_uint *buf_len, zend_serialize_data *data TSRMLS_DC);
        int (*unserialize)(zval **object, zend_class_entry *ce, const unsigned char *buf, zend_uint buf_len, zend_unserialize_data *data TSRMLS_DC);

        zend_class_entry **interfaces; /* 类实现的接口 */
        zend_uint num_interfaces; /* 类实现的接口数目 */

        char *filename; /* 类所在的文件 */
        zend_uint line_start; /* 类定义的开始行 */
        zend_uint line_end;/* 类定义的结束行 */
        char *doc_comment;
        zend_uint doc_comment_len;

        struct _zend_module_entry *module;/* 类所在的模块入口*/
    };


###参考文献：

- [PHP Internals Book](http://www.phpinternalsbook.com/)
- [深入理解PHP内核: Thinking in PHP Internals](http://www.php-internals.com/)
- [Extending and Embedding PHP](#)


[PHP扩展开发 - 构建第一个PHP扩展]: http://aicode.cc/article/397.html
