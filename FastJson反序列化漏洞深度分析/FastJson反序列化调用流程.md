# FastJson反序列化漏洞

介绍：

Fastjson 是由阿里巴巴开发的一个高性能 Java JSON 序列化/反序列化库，广泛用于将 Java 对象与 JSON 字符串相互转换。

1、反序列化漏洞的定义：

反序列化：
将 JSON 字符串还原为 Java 对象的过程。

漏洞本质：当 Fastjson 在处理不受信任的 JSON 数据时，攻击者可通过构造恶意 payload 触发任意代码执行（RCE），因为反序列化过程可能实例化攻击者控制的类。
AutoType 机制：Fastjson 的 AutoType 功能允许通过 @type 字段指定目标类，若未严格限制，攻击者可利用现有类路径中的“gadget”（工具类）执行恶意代码。

2、漏洞成因：
默认 AutoType 启用：

早期版本（如 1.2.24 及以下）默认启用 AutoType，允许不受限的类型解析。
黑名单机制不足：Fastjson 使用黑名单阻止危险类，但攻击者不断发现新 gadget（如 javax.swing.JEditorPane、java.lang.Thread），黑名单难以跟上。
全局配置风险：Fastjson 使用全局 ParserConfig 实例，若一个开发者启用 AutoType，可能会影响其他代码路径，导致全局漏洞。
绕过限制：即使 AutoType 被禁用，某些版本（如 1.2.80 及之前）仍可通过特定条件绕过限制。

3、典型案例与时间线
早期漏洞（2017 年起）：

● Fastjson 1.2.24 及之前版本存在反序列化漏洞，攻击者可通过 @type 指定恶意类执行代码。

● 例如，构造 {"@type":"java.lang.Runtime","value":"getRuntime().exec('calc')"} 弹出计算器。

创建一个User类:


    public class User {

    private int age;
    private String name;


    public int getAge() {
        System.out.println("调用了getAge方法");
        return age;
    }


    public String getName() {
        System.out.println("调用了getName方法");
        return name;
    }


    public void setAge(int age) {
        this.age = age;
        System.out.println("调用了setAge方法");
    }


    public void setName(String name) {
        this.name = name;
        System.out.println("调用了setName方法");
    }


    // toString 
    @Override
    public String toString() {
        return "User{name='" + name + "', age=" + age + "}";
    }}

使用parseObject，将json字符串反序列化

<img width="855" height="339" alt="image" src="https://github.com/user-attachments/assets/8fa73d3a-7a39-4d8e-ae67-0edb62dc22b4" />

<img width="342" height="108" alt="image" src="https://github.com/user-attachments/assets/3294cdc2-2691-4ac1-a410-c06b21acc294" />

parseObject会自动调用setter方法

使用@type来指定加载类
<img width="1011" height="288" alt="image" src="https://github.com/user-attachments/assets/e889f4ec-23d9-4cbd-842c-382839f0da83" />

<img width="408" height="174" alt="image" src="https://github.com/user-attachments/assets/0277370c-a375-4c95-8fbf-ff1cbaba94e7" />

会调用该类的set、get方法，

产生原因：

在parseObject反序列化的时候会调用目标类的set和get方法，
通过JSON字符串中的@type指定加载恶意类，
如果@type指定的这个类中的set或get方法包含命令执行，parseObject反序列化的时候就会执行方法中的命令。

示例：

创建一个evil类,
将evil类中的set方法写入Runtime.getRuntime.exec(calc)弹出计算器的命令

<img width="2081" height="1458" alt="image" src="https://github.com/user-attachments/assets/37bab96d-a617-451d-84ed-41de648d1982" />

使用@type来指定evil类，并设置参数，
最后利用parseObject来对Json字符串进行反序列化

<img width="1079" height="294" alt="image" src="https://github.com/user-attachments/assets/dc924c92-afc7-4ee9-8154-d3aeb16cbe14" />

成功加载了恶意类的set方法，并弹出计算器。

<img width="1427" height="702" alt="image" src="https://github.com/user-attachments/assets/4b0a02f8-6617-4cd4-8bf8-67cdc485bdb0" />

分析调试：

在parseObject处打上断点

<img width="1214" height="612" alt="image" src="https://github.com/user-attachments/assets/4cbc9a75-6686-4f8e-8785-0f3ffa1786b8" />

进入到parse

<img width="1449" height="696" alt="image" src="https://github.com/user-attachments/assets/783a8551-509d-44f8-a261-984593190e93" />

136行指定一个解析器DefaultJSONParser，主要是用来解析传入的Json字符串,
这里是Fastjson反序列化过程中的核心解析器

<img width="1446" height="882" alt="image" src="https://github.com/user-attachments/assets/f9312bed-babb-4fe6-82ff-36480423c19a" />

之后137行，调用parser去解析,
跟进parser

这个方法是处理JSON值的入口方法，根据JSONLexer的token类型来决定解析的策略,
parse()方法首先是初始化了一个JSONLexer,
使用switch语句，根据lexer.token()获取的当前token类型，解析JSON数据并返回对应的JAVA对象,

我们在启动类parseObject中传入的是{"@type":"evil","calc":"johnny"}
所以我们的代码会进入到1325行LBRACE中,
调用parseObject(object,fieldName)解析对象,
返回JSONObject

<img width="2559" height="1527" alt="image" src="https://github.com/user-attachments/assets/db65e19b-589f-42b3-b24a-1f4ad8a37967" />

进入parseObject,
在237行定义了一个key，之后用if判断来匹配字符，
我们当前是双引号" ,所以就会去匹配双引号中的字符，也就是@type

<img width="2559" height="1527" alt="image" src="https://github.com/user-attachments/assets/fefaa340-5eb2-4ba1-b809-660eed3943d3" />

随后在320行,
会判断，获取的key是否等于@type

<img width="1059" height="465" alt="image" src="https://github.com/user-attachments/assets/2bf7e0cc-97e2-4f8d-a7b3-fd7769c0ec91" />

<img width="1980" height="273" alt="image" src="https://github.com/user-attachments/assets/2be77bbe-56f0-45dd-bec9-1314a883f515" />

我们这里就是@type。

之后就会去获取@type后面的值，也就是@type指向的类,
赋值给typeName，然后利用loadClass加载这个类，也就是我们传入的evil类,
到这里解析Json字符串的部分就结束了。

下面就是反序列化的部分,
代码走到367行,
首先获取反序列化器再根据获取的反序列化器进行反序列化

<img width="2559" height="1527" alt="image" src="https://github.com/user-attachments/assets/a3bd0a73-c6b7-45ce-b531-4775b3d52e80" />

跟进

<img width="1452" height="942" alt="image" src="https://github.com/user-attachments/assets/1bb75b1f-d1a4-4218-a4e4-b80257bf340e" />

再次跟进，来到327行的getDeserializer方法,
这个方法是获取特定类反序列化器的核心方法,
主要功能是从缓存中获取现有的反序列化器，如果没缓存，就创建或者选择一个合适的反序列化器。

<img width="2559" height="1527" alt="image" src="https://github.com/user-attachments/assets/6af1a3b3-d034-4683-ba69-2a1ab1ec2de0" />

在362行对denyList黑名单做了遍历,
匹配是否存在Thread，这里禁止了线程相关的类

<img width="2559" height="1527" alt="image" src="https://github.com/user-attachments/assets/449deda6-3996-436f-b35c-28cbbf53fda5" />

往下走就是对一些特殊类型做处理，比如AWT类型，JDK8时间类型。。。。

<img width="1560" height="867" alt="image" src="https://github.com/user-attachments/assets/3d1ea18c-12e3-44dd-8c2f-646871e6998f" />

在前面的代码中都没匹配到,
代码走到461行,
创建一个JavaBeanDeserializer

<img width="1884" height="924" alt="image" src="https://github.com/user-attachments/assets/70538e66-4c90-4387-ad22-b17ed3b3814e" />

跟进来到526行,
调用了JavaBeanInfo.build

<img width="1767" height="893" alt="image" src="https://github.com/user-attachments/assets/51976873-52b8-46df-8ecd-c16306275ea6" />

它的作用就是，在创建传入类的反序列化器的时候，
通过反射分析目标类(clazz)的结构，构建JavaBeanInfo对象，
提取目标类的信息，也就是提取构造函数，getter、setter方法的信息

<img width="2559" height="1527" alt="image" src="https://github.com/user-attachments/assets/47f23f3c-44ae-4430-b44f-8f1fee854a5b" />

提取完之后,
会进入到3个for循环遍历

第一个for循环遍历：

遍历Method数据，提取符合规范的setter方法。
在代码设置了过滤条件

1、方法名小于4个字符的跳过（因为setter方法至少4个字符，比如setX）

2、跳过静态方法

3、确保返回值为void（标准的setter方法就是void）

4、确保参数个数为1 （JavaBean setter接收单一参数）

5、第五个字符需要是大写 （符合命名规范）

最后添加到FieldInfo

<img width="2331" height="1461" alt="image" src="https://github.com/user-attachments/assets/7c9ad1d3-341d-4d64-961b-597f98cccc7b" />

第二个for循环遍历：

专门用来处理返回的字段
遍历上一个for循环生成的FieldInfo，提取字段。

1、跳过了static字段，因为与实例无关

2、如果字段是final，就检查是否支持以下几种类型

3、检查fieldList中是否已有同名的FieldInfo

<img width="2214" height="1460" alt="image" src="https://github.com/user-attachments/assets/57002dc8-526d-45d7-92b1-4051cbd8657d" />

第三个for循环遍历：

遍历clazz.getMethod返回的public方法，识别以get开头的getter方法，生成FieldInfo对象。

并与前两个for循环的结果合并，通过return new JavaBeanInfo构造JavaBeanInfo实例。

<img width="2214" height="1460" alt="image" src="https://github.com/user-attachments/assets/014bcf53-fff7-476e-9007-3941313c16c9" />

JavaBeanInfo实例构造完成之后，获取到了反序列化器，代码回到这里,
调用获取到的反序列化器deserializer.deserialze对clazz进行反序列化

<img width="2330" height="1460" alt="image" src="https://github.com/user-attachments/assets/0fcfdd5f-bf23-417b-9c7a-8efc43093aaa" />

跟进反序列化方法,

层层调用
最后走到FieldInfo类下的449行get方法，调用了invoke反射执行了evil

<img width="2559" height="1527" alt="image" src="https://github.com/user-attachments/assets/2e5ec4e0-b108-407a-9080-c9b36a2e903b" />


参考连接：
https://y4er.com/posts/fastjson-learn/

https://www.bilibili.com/video/BV1pP411N726/






























