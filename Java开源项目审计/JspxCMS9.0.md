该项目版本存在

1、SQL注入

2、SSRF

3、Shiro1.3.2反序列化

4、任意文件上传

SSRF漏洞：

SSRF 漏洞出现的场景有很多，如在线翻译、转码服务、图片收藏/下载、信息采集、邮件系统或者从远程服务器请求资源等。
通常我们可以通过浏览器查看源代码查找是否在本地进行了请求，也可以使用 DNSLog 等工具进行测试网页是否被访问。
但对于代码审计人员来说，通常可以从一些 http 请求函数入手，审计 SSRF 漏洞时需要关注的一些敏感函数

HttpClient.execute()

HttpClient.executeMethod()

HttpURLConnection.connect()

HttpURLConnection.getInputStream()

URL.openStream()

URL.openConnection()

openConnection()

HttpServletRequest()

BasicHttpEntityEnclosingRequest()

DefaultBHttpClientConnection()

BasicHttpRequest()

urlConnection.getInputStream

OkHttpClient.newCall.execute

Request.Get.execute

Request.Post.execute      

ImageIO.read

SSRF漏洞1：

全局搜索HttpClient.execute关键函数
来到 src/main/java/com/jspxcms/ext/domain/Collect.java 94行 ftchHtml方法

<img width="1028" height="735" alt="Image" src="https://github.com/user-attachments/assets/4d1bb878-412e-4f41-932b-5b74f29fc2f2" />

在97行使用了 HttpClient.execute 传入了httpget 
而httpget接受了uri

跟踪fetchHtml方法 查看是谁调用了它
发现是上面一个fetchHtml方法 在最后return返回了这个fetchHtml方法

<img width="821" height="735" alt="Image" src="https://github.com/user-attachments/assets/b0d48ee0-c312-4de1-a1b5-4561779288d5" />

跟踪86行的fetHtml
来到 com/jspxcms/ext/web/back/CollectController.java   Controller层 244行 fetchUrl方法

<img width="1022" height="735" alt="Image" src="https://github.com/user-attachments/assets/6b929d87-5794-485b-b3a5-fe2d519cc793" />

代码分析：

接收参数 url charset userAgent 

然后try catch捕获异常
定义soure 调用fetchHtml 传入 url charset userAgent

我们在前面知道了fetchHtml方法使用了 HttpClient.execute 方法 
而这里的fetchUrl调用了fetchHtml，并且接收了url传参
而且这里的url是我们可以控制的
那么就可以造成SSRF漏洞

来到CollectController类最顶端查看定义的路由
是/ext/collect    在fetchUrl定义的是fetch_url.do 然后接收传参 url

<img width="790" height="373" alt="Image" src="https://github.com/user-attachments/assets/7f09b668-0828-47ce-9874-2fb31912c9df" />

在项目后台的路径都是以cmscp开头的

那么构造payload：
/cmscp/ext/collect/fetch_url.do?url=http://dns.log

<img width="580" height="97" alt="Image" src="https://github.com/user-attachments/assets/186920bb-ccd5-43d9-b255-f1765ecb1b3f" />

<img width="580" height="97" alt="Image" src="https://github.com/user-attachments/assets/84a81f91-ffd7-4b14-bc8e-3729379b772e" />

验证成功，存在SSRF漏洞

SSRF漏洞2：

全局搜索openConnection函数

<img width="851" height="432" alt="Image" src="https://github.com/user-attachments/assets/695e698c-3a71-42a4-b7ef-8a787195806b" />

来到 src/main/java/com/jspxcms/core/web/back/UploadControllerAbstract.java 120行 ueditorCatchImage方法

<img width="1014" height="735" alt="Image" src="https://github.com/user-attachments/assets/3cc8ad75-8686-4180-94b8-adbbf64ec5d9" />

在144行 使用了 openConnection
传入了src
src是136行的一个 source
网上看131行 是定义了一个用来接收前端传参的source[]参数 说明等会构造payload时 source[]是一个关键的参数
全局搜索ueditorCatchImage方法 发现有两处调用了这个方法

<img width="979" height="305" alt="Image" src="https://github.com/user-attachments/assets/2c8aecc9-41cd-494c-899c-b082733c870f" />

一个是 src/main/java/com/jspxcms/core/web/fore/UploadController.java 
另一个是 src/main/java/com/jspxcms/core/web/back/UploadController.java
通过分析发现，这两个方法代码都是相差不大的  一个是前台，一个是后台调用
先分析一下 src/main/java/com/jspxcms/core/web/back/UploadController.java 后台调用
来到80行

<img width="1043" height="735" alt="Image" src="https://github.com/user-attachments/assets/9812f6b6-b59e-4649-b261-0af17728954a" />

往上看，是上面的ueditor方法中的66行使用了 ueditorCatchImage 

<img width="1043" height="735" alt="Image" src="https://github.com/user-attachments/assets/36a9678c-4474-4480-975f-6b5769576e33" />

并且调用时做了一个校验，else if ("catchimage".equals(action))
意思就是 如果action参数里面的值是catchimage，那么就调用ueditorCatchImage方法
我们需要做的就是 要将catchimage传入到action参数中，并且在之前的 source[]传入我们的dnslog

构造payload：
在ueditor方法中的路由是 /ueditor.do
网上翻上层路由是/core

<img width="946" height="448" alt="Image" src="https://github.com/user-attachments/assets/a1ad0dd3-4346-4ef0-a254-738a793af499" />

因为这是后台的类
所以在前面还要加上/cmscp
完整构造：
/cmscp/core/ueditor.do?action=catchimage&source[]=http://dnslog

<img width="635" height="175" alt="Image" src="https://github.com/user-attachments/assets/1c5b738d-0fb0-415b-9d35-8374c93effb0" />

<img width="566" height="274" alt="Image" src="https://github.com/user-attachments/assets/d4f9fa8e-cd95-4fb5-baf5-5b67a5aa7ed0" />

验证成功 存在SSRF漏洞

再来分析一下前台的src/main/java/com/jspxcms/core/web/fore/UploadController.java

<img width="1043" height="735" alt="Image" src="https://github.com/user-attachments/assets/aa0967e4-8b55-4a91-9150-260a75eb26d1" />

代码几乎是一摸一样
但是存在于前台，所以没有/cmscp/core路径

只有一个ueditor路由

payload：
/ueditor?action=catchimage&source[]=http://dnslog
<img width="589" height="182" alt="Image" src="https://github.com/user-attachments/assets/3dcf2f92-0ff5-42da-8780-d8237a93735c" />
<img width="589" height="182" alt="Image" src="https://github.com/user-attachments/assets/8a360d4a-615d-4199-acda-56b8e21663b9" />

shiro反序列化漏洞

在pom文件中 使用了shiro1.3.2版本

<img width="1043" height="735" alt="Image" src="https://github.com/user-attachments/assets/2bed1f1b-f2d9-41ac-829e-0d55b17a05ea" />

该版本存在shiro反序列化漏洞
但是查看一些文章，要对cookie爆破一个小时
懒得爆破了

文件上传

在网站后台处存在上传文件的功能

<img width="1280" height="500" alt="Image" src="https://github.com/user-attachments/assets/6d09767c-6e2e-4cf4-a3f3-9a3c683081dc" />

但是在上传文件处，上传jsp文件，并不能正常访问

<img width="788" height="750" alt="Image" src="https://github.com/user-attachments/assets/b30da48d-7257-44e1-9e6a-2047d61be85a" />

需要搭载tomcat启动项目  懒得搭了



模板注入漏洞：

${"freemarker.template.utility.Execute"?new()("calc")}
这是payload的意思就是
使用new一个freemarker.template.utility.Execute对象
来执行calc

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/4b68f6e0-c181-4390-9c88-dd17478a0ce6" />

在网站设置里修改模板

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/6237119d-ff86-4ba0-9a1b-cfb2c032fbcf" />

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/e87cd50f-0a55-41a2-87b7-7b756b925538" />
