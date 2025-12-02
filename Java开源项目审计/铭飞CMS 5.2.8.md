免费可商用的开源Java CMS内容管理系统/基于SpringBoot 2/前端Vue3/element plus/提供上百套模
板,同时提供实用的插件/每两个月收集issues问题并更新版本/一套简单好用开源免费的Java CMS内容管
理系/一整套优质的开源生态内容体系。我们的使命就是降低开发成本提高开发效率，提供全方位的企业
级开发解决方案。

该版本项目存在

1、Fastjson反序列化

2、shiro反序列化

3、freemaker模板注入

4、SQL注入

5、任意文件上传

6、任意文件删除

7、XSS

shiro反序列化漏洞：

在配置文件里看到有shiro-key

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/33867c85-7d14-4dce-8c42-a5bc98f54f40" />

工具直接跑

<img width="760" height="628" alt="Image" src="https://github.com/user-attachments/assets/d1346515-0f3f-4670-bc73-8558e5f1241b" />

存在shiro反序列化漏洞


freemaker模板注入漏洞：

在pom文件中并没有看到有freemaker的引入依赖  但是在readme文件中标注了 使用了Freemaker视图框架

<img width="1050" height="735" alt="Image" src="https://github.com/user-attachments/assets/7adea9d2-6009-43b8-aefa-c1bd92d284a4" />

来到后台模板管理处

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/762a8635-5335-41d6-bea6-33196d5cc0bc" />

在search.html处插入payload ： <#assign ex="freemarker.template.utility.Execute"?new()> ${ ex("calc") }

#assign是指令的意思
ex是一个变量名，后面是他的值
freemarker.template.utility.Execute"?new() 这个是ex的值
new()表示实例化一个类
实例化哪个类呢 就是freemarker.template.utility.Execute 中的Execute类
实例化之后我们就可以使用它的
然后使用ex来使用Execute来执行传入的值，也就是calc，达到命令执行的操作

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/9e66ee09-f563-42de-b9aa-209095e4fcd5" />

现在保存，然后访问search.html  在前端的搜索功能

<img width="1280" height="770" alt="Image" src="https://github.com/user-attachments/assets/2b149f09-816c-40f2-ad4a-1f5adf3ae8e2" />
