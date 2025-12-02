一、项目简介

java 版CMS系统、基于java技术研发的内容管理系统、功能：栏目模板自定义、内容模型自定义、多个站点管理、在线模板页面编辑等功能、代码完全开源、MIT授权协议。
技术选型：jfinal DB+Record mysql freemarker Encache spring 等 layui zTree bootstrap 。特点：支持多站点、可以根据需求添加手机站、pc站。
项目地址：https://gitee.com/oufu/ofcms

二、漏洞挖掘

首先检查pom文件

log4j-1.2.16 远程代码执行漏洞

fastjson-1.1.41 反序列化漏洞

freemarker 模板注入漏洞

shiro1.3.2 该版本存在未授权漏洞


1.log4j远程代码执行漏洞

该项目中 pom.xml 文件中引入了 log4j 1.2.16 版本，该版本存在反序列化漏洞，其 CVE 编号为 CVE2019-17571。
Log4j1.2.x 版本中包含一个 SocketServer 类，在未经验证的情况下，该SocketServe类很容易接受序列
化的日志事件并对其进行反序列化，在结合反序列化工具使用时,可以利用该类远程执行任意代码。
但该漏洞对于 jdk 限制为 Jdk7u21，如果大于该版本则可能无法利用该漏洞。


2.fastjson反序列化漏洞

全局搜索关键字 parse，parseObject
发现在src/main/java/com/ofsoft/cms/core/utils/okhttp/OkHttp.java处使用了parseObject

<img width="1966" height="1470" alt="image" src="https://github.com/user-attachments/assets/a5a311ac-2154-462d-8480-7a5397979e0f" />

但是并没有传参点，所以无法利用


3.freemarker SSTI模板注入漏洞

该项目使用的是freemarker2.3.21，该版本存在SSTI模板注入漏洞
<img width="801" height="136" alt="image" src="https://github.com/user-attachments/assets/dde326fc-1fc2-4a7f-815f-67e7ccb1d037" />

来到网站后台，模板文件处

<img width="1569" height="1498" alt="image" src="https://github.com/user-attachments/assets/08ce29e9-97af-429d-9805-0ce003be540d" />

在文件里插入payload ：<#assign ex="freemarker.template.utility.Execute"?new()> ${ ex("calc") }

<img width="1554" height="1359" alt="image" src="https://github.com/user-attachments/assets/ed2b72a9-1578-4826-9bef-a4334d4a41a6" />

再去访问index页面

<img width="1569" height="1498" alt="image" src="https://github.com/user-attachments/assets/8ca9021d-30ec-43c1-89db-160f508c3a21" />

弹出计算器，存在SSTI模板注入漏洞

获取保存时的数据包

<img width="1884" height="1491" alt="image" src="https://github.com/user-attachments/assets/47e6df5c-774d-4b29-bbf3-ff0c1e126099" />

路由指向/ofcms-admin/admin/cms/template/save.json

全局搜索/admin/cms/template

来到com/ofsoft/cms/admin/controller/cms/TemplateController.java

<img width="2302" height="1470" alt="image" src="https://github.com/user-attachments/assets/ef799e55-812a-49e3-9be7-7379e20a6d2e" />

在121行处 getRequest().getParameter，使用file_content获取我们传入的文件数据，

<img width="1884" height="1491" alt="image" src="https://github.com/user-attachments/assets/3b8fb06f-b92e-4e02-b2dd-c6f996598bf6" />

然后将文件内容中的&lt替换为<   ,   &gt替换为>,也就是html解码

之后再new一个file来接收传入的fileName，和pathFile

最后再writeString写入file文件和Content

payload为<#assign value="freemarker.template.utility.Execute"?new()>  ${ value("calc") }

其中 #assign为指令

value是一个变量名，后面是他的值

freemarker.template.utility.Execute"?new() 就是value的值，new就是实例化

这个payload的意思就是 利用assign指令，将freemarker.template.utility.Execute 给实例化，最后利用Execute类来执行 value中的 calc 命令

4.任意文件写入漏洞

同样 在模板保存这段代码中
<img width="1729" height="1470" alt="image" src="https://github.com/user-attachments/assets/96a36905-8774-4bd6-9a49-969fc3c13451" />

分析代码可以发现，这段代码中的用来接收文件数据的变量fileContent没有对文件内容进行任何的过滤。
在fileName中也是，使用的getPara(file_name)，对传入的文件名也没有做任何的处理。

根据IDEA断点调试，这个index.html位于我D:\apache-tomcat-9.0.100\webapps\ofcms-admin\WEB-INF\page\mobile
<img width="2559" height="1539" alt="image" src="https://github.com/user-attachments/assets/963bc179-0b58-47a0-ba00-8999e74970bd" />

现在要写入一个webshell到static静态目录，并访问
该项目静态目录位于
D:\apache-tomcat-9.0.100\webapps\ofcms-admin\static
从WEB-INF\page\mobile到static总共三个目录，那么构造payload文件名就应该是
../../../static/shell.jsp

将filename修改为../../../static/shell.jsp
file_content改为url编码后的webshell
<img width="1884" height="1491" alt="image" src="https://github.com/user-attachments/assets/f3e0ff8d-ba8b-42ac-8c74-a568a27c8d7c" />

<img width="1245" height="403" alt="image" src="https://github.com/user-attachments/assets/fc904e05-6569-4648-a35f-8beef7de815f" />

<img width="733" height="780" alt="image" src="https://github.com/user-attachments/assets/85f710b8-5dc0-433d-bceb-808d247b273a" />

5.SQL注入漏洞

<img width="1575" height="1498" alt="image" src="https://github.com/user-attachments/assets/21616866-abbe-4331-a958-f7237ed78ee5" />

在后台代码生成-》创建表 处可以创建SQL语句

<img width="1884" height="1491" alt="image" src="https://github.com/user-attachments/assets/00483de9-4b63-4b74-972e-cf10f9d408d5" />

跟踪路由
/ofcms-admin/admin/system/generate/create.json

<img width="2304" height="1470" alt="image" src="https://github.com/user-attachments/assets/30c54efa-de06-4aa7-8b47-63a1b3ddcb0c" />

来到src/main/java/com/ofsoft/cms/admin/controller/system/SystemGenerateController.java crate方法

首先是使用了getPara来获取sql
然后使用了DB.update来对传入的sql做处理

<img width="1986" height="1470" alt="image" src="https://github.com/user-attachments/assets/3bac0334-8397-450b-8904-22e80c38c462" />

这是内置在jfinal中的方法 使用了 pareparseStatement预编译的方式来处理传入的sql语句
但是我们在这里是输入一整条sql语句 并不触发预编译的 （?.?.?）占位符

payload：update of_cms_link set link_name=updatexml(1,concat(0x7e,(user())),0) where link_id = 4

<img width="1479" height="1500" alt="image" src="https://github.com/user-attachments/assets/9c83aa06-c846-473a-ac65-c7971f379f3c" />

<img width="1728" height="831" alt="image" src="https://github.com/user-attachments/assets/3f67287c-9ee2-42c2-95cc-81e78e165863" />
