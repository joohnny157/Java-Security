项目存在漏洞：

1、H2 Database代码注入漏洞(CVE-2021-42392)

2、任意文件删除漏洞

3、任意文件写入漏洞

4、任意文件读取漏洞

5、Freemarker模板注入漏洞

6、SSRF漏洞

7、远程命令执行漏洞

H2 Database代码注入漏洞(CVE-2021-42392)
检查pom文件发现引入了H2 Database

<img width="701" height="397" alt="Image" src="https://github.com/user-attachments/assets/ac0bd517-1e1d-4586-8e9e-05a73b2e7c4c" />

该依赖存在CVE漏洞

<img width="334" height="163" alt="Image" src="https://github.com/user-attachments/assets/7e65daaa-93bb-4c52-98a8-93a822c496ae" />

该漏洞主要是jndi注入

打开H2database项目进行分析

漏洞产生的主要点在于
src/main/org/h2/util/JdbcUtils.java   303行

<img width="1048" height="735" alt="Image" src="https://github.com/user-attachments/assets/8286db75-db8d-4b14-82de-5ec6a5f2b95d" />

首先是判断了d是否是javax.naming.Context类或其子类的实例，如果是，则继续往下面执行
使用context.lookup的方式来处理传入的url
而这里的d 来自290行

<img width="602" height="398" alt="Image" src="https://github.com/user-attachments/assets/b0f34f34-cec7-4b87-991c-ab8d5acc9824" />

使用loadUserClass加载自定义类driver
这个driver类就是在前端里的加载类

<img width="622" height="367" alt="Image" src="https://github.com/user-attachments/assets/ca02949a-ac9c-4f6d-873d-635fa7899715" />

而这里的 javax.naming.InitialContext类 是javax.naming.Contex的子类
从而进入到 else if下面的try

<img width="810" height="735" alt="Image" src="https://github.com/user-attachments/assets/47fa7efc-b744-41c5-b8d6-3514b11c6a99" />

调用链：

1、在303行中的context.lookup(url)  在getConnection中，来看一下getConnetction方法的调用

<img width="810" height="735" alt="Image" src="https://github.com/user-attachments/assets/9db16033-f022-4a3d-959b-09af0bbf95fe" />

根据栈调用顺序，先来看一下Webserver类

<img width="810" height="735" alt="Image" src="https://github.com/user-attachments/assets/32f34935-de64-4063-8889-46a5031dc4ff" />

这个getConnection设置接收形参 driver，databaseUrl，user，password
这个方法是用来建立数据库连接的
前面的方法是用来建立h2开头的数据库连接的，
但是我们要实现的是jndi注入 所以是ldap开头的
直接进入到最后一句代码
JdbcUtils.getConnection(driver, databaseUrl, p)

下面，查看谁使用了 WebServer 类下的 getConnection 方法

<img width="720" height="377" alt="Image" src="https://github.com/user-attachments/assets/08dc8a12-957e-4856-9ff2-c6d22eed656a" />

3、来到WebApp类下的login方法
在login方法中使用了attributes来获取四个属性 driver，url，user，password

之后在954行使用server.getConnection(driver, url, user, password)来建立连接

<img width="668" height="376" alt="Image" src="https://github.com/user-attachments/assets/9afb8717-87cc-4848-a9af-2a0763f3fb66" />

4、跟进login

<img width="610" height="485" alt="Image" src="https://github.com/user-attachments/assets/b880fc03-de3b-4d4e-ab60-f1c5ef321a2c" />

来到WebApp类的207行 process方法
首先定义了一个形参file
完后while循环 在211行调用了login()方法
判断传入的file是否等于login.do，然后直接调用login方法

<img width="453" height="286" alt="Image" src="https://github.com/user-attachments/assets/b6f5d62b-e7ac-4fef-a3c9-c3d6dca1c3ec" />

5、跟进process
<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/9ff2d803-0971-4c56-b090-a859632b845a" />

是WebApp中的processRequest方法中的170行调用了 process
在169行做了一个if判断，如果file是一.do结尾的
那么就调用process里面传入file

6、跟进processRequest

<img width="810" height="735" alt="Image" src="https://github.com/user-attachments/assets/bd549c3b-769e-4bce-a3ac-509329220fa5" />

来到WebThread中的process方法
这个方法用来处理http请求，判断主机是否存活，是否是GET请求或POST请求

在134行使用了processRequest传入两个参数 file，hostAddr

7、继续跟进process方法

来到WebThread85行run()方法

<img width="810" height="735" alt="Image" src="https://github.com/user-attachments/assets/4338f38a-a891-4bc8-a9b4-eb9e87ac0685" />

在90行对process做了一个判断
显示while循环嵌套if判断
不断调用process来处理客户端请求
一直循环，直到不是process为false止




任意文件删除漏洞：

在后台 设置-博客备份-文章备份处存在删除功能

<img width="1280" height="700" alt="Image" src="https://github.com/user-attachments/assets/18803f4d-b258-4007-b09c-bedde8c997fe" />

<img width="881" height="708" alt="Image" src="https://github.com/user-attachments/assets/d193d799-847b-448c-9fe6-725565a09c7f" />

根据关键字delBackup定位代码
来到/web/controller/BackupController下的delBackup方法

<img width="1280" height="595" alt="Image" src="https://github.com/user-attachments/assets/1920f200-e544-4b55-97ff-95d75dda06be" />

代码分析：
首先接收两个形参 filename type

System.getProperties().getProperty("user.home") 用来获取用户家目录 + /halo/backup + type + filename 
之前数据包中的type类型是posts
那么文件的路径就是 C:/user/username/halo/bachup/posts/filename.zip
构造payload尝试删除C:/TEST/test.txt
../../../../../TEST/test.txt

<img width="881" height="708" alt="Image" src="https://github.com/user-attachments/assets/71823445-8791-4ffe-90ad-8223bff60eb5" />

成功删除，存在任意文件删除漏洞

任意文件写入漏洞：

在前端主题编辑处

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/29ce4cef-d8d6-4a7e-940b-6bed6b469d4e" />

可以自定义文件名和内容

<img width="881" height="708" alt="Image" src="https://github.com/user-attachments/assets/16ca6098-e77b-48f9-aaa0-2e3b2dc20774" />

搜索关键字 来到cc/ryanc/halo/web/controller/admin/ThemeController.java 下的savaeTpl方法中

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/c722f7d6-a228-4390-ac30-26327ed7fe66" />

首先接收两个形参 tplName和tplContent
if判断传入的传入的tplContent是否为空 如果为空就return错误信息
如果不为空就继续下面的代码
new一个file对象 ，ResourceUtils.getURL("classpath:").getPath()用于获取项目根路径
然后获取主题路径
然后使用Hutool中的FileWriter方法将文件写入到tplPath中
在这期间并对文件的内容，文件写入的路径和文件名做过滤

<img width="881" height="708" alt="Image" src="https://github.com/user-attachments/assets/1aa63567-c023-437a-8cdb-b8ad6329554a" />

我的盘符是从D开始的
D盘到项目根路径14个目录 所以是14个../ 

<img width="1139" height="699" alt="Image" src="https://github.com/user-attachments/assets/21220709-f5ad-440a-8f24-b7ca022e0916" />

写入成功 存在任意文件写入漏洞

任意文件读取漏洞：

存在于cc/ryanc/halo/web/controller/admin/ThemeController.java 下的getTplContent方法

<img width="1244" height="750" alt="Image" src="https://github.com/user-attachments/assets/f5ee34c1-37a4-4693-80d3-a6bf58796f0f" />

接收形参tplName
首先获取项目根路径 然后获取主题路径
使用Hutool中的FileReader读取tplName传入的文件
然后以readString方式读取
最后return返回String格式的文件内容
在此期间并没有对文件名，文件路径，文件内容做限制和过滤
导致攻击者可以构造特殊的路径和文件名来读取服务器文件
比如../../../etc/passwd

<img width="881" height="708" alt="Image" src="https://github.com/user-attachments/assets/456cb763-f31a-4dc7-938b-38e5a0b16a33" />

成功读取到D盘下的1.txt文件  存在任意文件读取漏洞

SSRF漏洞：

在后台外观-主题处
存在一个安装主题
在安装主题处存在远程拉取功能

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/2fa73167-86f3-47e1-b8e7-01120e29b94b" />

定位接口 来到cc/ryanc/halo/web/controller/admin/ThemeController.java 中的 clone方法

<img width="881" height="708" alt="Image" src="https://github.com/user-attachments/assets/2e22d0e6-955a-4bb6-a247-342d7540f49c" />

<img width="881" height="708" alt="Image" src="https://github.com/user-attachments/assets/0943c383-7947-4671-84a5-8bc632f27fb2" />

接收两个形参 一个remoteAddr 一个 themeName

首先if判断这两个形参传入的值是否为空
如果为空则返回错误信息
如果不为空，继续下面的代码
basePath 获取项目根路径
themePath获取主题路径
关键点在于190行 使用RuntimeUtil.execForStr来拼接传入的remoteAddr+主题路径+传入的themeName
execForStr是Hutool的一个工具类用来执行命令
这里的git clone就是执行的命令

<img width="882" height="373" alt="Image" src="https://github.com/user-attachments/assets/f12981af-c145-4a35-9a3f-a37a84893cba" />

造成ssrf漏洞的原因是 使用了git clone来获取主题，但是这里的remoteAddr是由我们在前端可以控制的
