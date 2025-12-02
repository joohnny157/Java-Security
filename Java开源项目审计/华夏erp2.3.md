管伊佳ERP（原名华夏ERP）基于SpringBoot框架和SaaS模式，立志为中小企业提供开源好用的ERP软
件，目前专注进销存+财务+生产功能。主要模块有零售管理、采购管理、销售管理、仓库管理、财务管
理、报表查询、系统管理等。支持预付款、收入支出、仓库调拨、组装拆卸、订单等特色功能。拥有商
品库存、出入库统计等报表。同时对角色和权限进行了细致全面控制，精确到每个按钮和菜单。


该版本项目存在

1、Fastjson反序列化漏洞

2、SQL注入漏洞

3、权限绕过漏洞

4、越权漏洞

5、XSS漏洞

SQL注入漏洞

全局搜索${ 检查是否存在拼接SQL语句

<img width="821" height="432" alt="Image" src="https://github.com/user-attachments/assets/7a643ca1-d92d-4ae6-81d3-87223b2fffee" />

搜索发现存在很多
来到mapper层 
src/main/resources/mapper_xml/UserMapperEx.xml

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/c3ab9ce7-553f-45c5-bbff-6e9e06a23877" />

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/d52e66eb-05eb-4e7e-8bd3-250ee0464fdf" />

传入两个参数 userName，loginName
继续跟进

来到UserController

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/3ce21881-bd5a-4ee4-8597-74834687a41a" />

接口位于/addUser
来到后台抓包

<img width="863" height="699" alt="Image" src="https://github.com/user-attachments/assets/d03c95b9-7b9b-4fd5-a377-182f257cb73a" />

<img width="863" height="699" alt="Image" src="https://github.com/user-attachments/assets/00774725-8501-4f1e-97f3-8d5acd18e639" />

用/user/list?search= 这个接口跑sqlmap跑不出来

<img width="876" height="468" alt="Image" src="https://github.com/user-attachments/assets/24bb52f5-1c07-43c1-8fb3-a03e84b6f867" />

但是用  /unit/list?search可以跑出来
在search={"name":"*"}  让sqlmap跑
<img width="876" height="468" alt="Image" src="https://github.com/user-attachments/assets/2113f359-96cc-4b9a-9944-74e833b6ba30" />

后续测试发现 只有name可以用sqlmap跑出来  userName和loginName都没成功过


Fastjson反序列化漏洞：

检查pom文件 使用了fastjson1.2.55版本 存在反序列化漏洞

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/5b8fab4c-3d83-4a27-a66d-7acb25e03933" />

全局搜索 parse和parseObject

<img width="821" height="432" alt="Image" src="https://github.com/user-attachments/assets/9a48a4c1-65ea-44bc-8f07-0ad9995b2df6" />

search参数是我们可控的

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/812d0682-16e8-453a-9290-848888b2a665" />

接口位于/material/getMaterialEnableSerialNumberList

<img width="878" height="416" alt="Image" src="https://github.com/user-attachments/assets/d745efe7-264d-4798-9106-6e6a805763b5" />

位于后台 序列号 批量添加

<img width="788" height="749" alt="Image" src="https://github.com/user-attachments/assets/b4ac798a-6c08-4686-a40c-1d56f3f7fbfe" />

在search处插入payload ： {"@type":"java.net.Inet4Address","val":"0rcsoq.dnslog.cn"} 并将其url编码

<img width="863" height="699" alt="Image" src="https://github.com/user-attachments/assets/e53721b5-6436-4122-b5db-d0dc0dbc97cd" />

dnslog成功接收请求  存在fastjson反序列化

<img width="1218" height="674" alt="Image" src="https://github.com/user-attachments/assets/b7eb1df1-a324-48f7-8e6b-e7ab6179a697" />

权限绕过漏洞：

直接来到filter层

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/b691305c-ff3a-473f-aa13-ca8473df68dd" />

代码分析
48行 使用了 servletRequest.getRequestURI 来获取URL路径，并没有对URL路径进行一个过滤或者校验，会造成目录穿越

userInfo 使用 servletRequest.getSession()获取用户Session身份信息

if判断 如果 userInfo不为空 那么就是已经登录，不会阻止访问路径

if判断，如果requestUrl 不为空 并且 url路径中包含了 /doc.html   /register.html  /login.html
那么就放行 不会阻止访问  

在前面的requestUrl中使用的是servletRequest.getRequestURI 没有对URL进行../的过滤

那么在55-58这几行代码中描述的是 
如果url路径中存在/doc.html  /register.html  /login.html 中的一个，那么就会放行

那如果在这三个之后加上../  再加上后台路径 就可以成功绕过session校验

漏洞验证：

先随便抓个后台数据包

<img width="863" height="699" alt="Image" src="https://github.com/user-attachments/assets/00233950-df90-4ff1-b1ea-133f43a517be" />

删除cookie  返回302

<img width="863" height="699" alt="Image" src="https://github.com/user-attachments/assets/0cb0ffb3-de02-4acc-803b-9cb16bc97530" />

在路径前加上/login.html 再加上../

<img width="863" height="699" alt="Image" src="https://github.com/user-attachments/assets/46fbf0a6-6b45-4e90-a82f-cf5b0311a7cb" />


越权漏洞：

位于 UserController类中
这个Controller是用来对user信息做增删改查的

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/7a59a5c0-6326-4827-81cc-9ad31a0ea474" />

先分析一下resetPwd 重置密码这个方法

<img width="662" height="238" alt="Image" src="https://github.com/user-attachments/assets/66af839a-2b73-4edf-a88a-0dcfed8ec44a" />

RequestParam接收参数 id  
197行 password=123456   意思就是重置密码为123456
然后md5 加密
最后使用 userService.resetPwd(md5Pwd,id)
在update更新密码中 只传入了md5Pwd和id两个参数
并没有其他的校验，比如cookie，而且这个项目并没有使用其他的安全框架，只写了filter，而filter只有一些路径过滤

跟进一下resetPwd来到UserService类

<img width="815" height="735" alt="Image" src="https://github.com/user-attachments/assets/81e5ac5b-2de6-42fc-a061-4ae7b438bcb3" />

这里重置密码是根据id来重置密码的


来到后台用户管理处

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/bbba57b4-d81e-486e-8b63-6e59c62c2e23" />

重置密码抓包，数据包中只有一个id参数

<img width="863" height="699" alt="Image" src="https://github.com/user-attachments/assets/76e45121-e5bb-47bb-b536-3a0e6742e684" />

在UserController中所有方法都存在一样的问题

现在三个用户的密码都是123456
现在来尝试越权修改密码

三个ID 63,120,131   

<img width="1050" height="735" alt="Image" src="https://github.com/user-attachments/assets/7fa8d119-9e66-4de4-acf1-b1c46813bf7d" />

现在尝试一下修改测试用户的密码

<img width="863" height="699" alt="Image" src="https://github.com/user-attachments/assets/5f20ebad-d404-40a2-9fe5-24130950a557" />

抓包修改ID 直接可以修改其他用户的密码
回到DataGrip刷新

<img width="1050" height="735" alt="Image" src="https://github.com/user-attachments/assets/82f2455d-4253-474f-8190-0729fb29f461" />







