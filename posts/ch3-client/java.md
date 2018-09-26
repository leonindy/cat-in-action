## Java语言集成

- **这是一个新项目如何接入cat-client的指南**
- **完成 1、2 步骤，集成CAT的基本工作已完成**
- **完成 3、4、5 步骤配置会大大提升你使用体验，强烈建议完成所有接入步骤**

### 1. pom.xml依赖配置

#### 特殊项目使用pom依赖
```
<dependency>
    <groupId>com.dianping.cat</groupId>
    <artifactId>cat-client</artifactId>   
    <version>${cat-version}</version>
</dependency>
```


- 请将cat-client提前部署在maven仓库。 

### 2. 项目名


    1. 在resources资源文件META-INF下，添加文件“app.properties”
    
       注意是src/main/resources/META-INF/这个文件夹，而不是webapps下的那个META-INF
    
    2. 在app.properties里加上：app.name=XXX
    
       例如你的项目名是broker-service，即可设置为app.name=broker-service
       
    3. 配置如下图所示


### 3. 环境依赖

#### 3.1 依赖的文件
- /data/appdatas/cat/client.xml 

    - 应用可读权限(如果此文件不存在，默认发送到线下环境的cat)
    
```
<config>
    <servers>
            <server ip="127.0.0.1" port="2280" />
    </servers>
</config>
```
#### 3.2 依赖的目录及权限

- /data/applogs/
    
	- 该目录必须要存在，并且是应用启动用户可读写 (R+W) 的权限
	- cat-client会自动创建/data/applogs/cat/cat_20160823.log这种格式log，cat相关日志会打到这里，cat初始化问题可以到这里查看

- /data/appdatas/
	- 该目录必须要存在，并且是应用启动用户可读写 (R+W) 的权限
	- cat-client会自动创建/data/appdatas/cat/目录，同时会生成${appKey}.mark的临时文件


``` 注：如上述目录不存在或权限有问题，cat会在/tmp/目录下创建相关路径和文件；但是如果/tmp会定时清理的话，对cat的使用有影响，如 ```[消息ID串掉问题](/faq/#4-id)

### 4. web相关配置

注：提供HTTP请求的应用使用，如果是一个TCP的服务请忽略
    
#### 4.1 添加CatFilter

如果项目是对外不提供URL访问，仅仅提供Pigeon服务，此项忽略. CatFilter作用

1. 一个http请求来了之后，会自动打点，能够记录每个url的访问情况，并将以此请求后续的调用链路串起来，可以在cat上查看logview
2. 可以在cat Transaction及Event 页面上都看到URL和URL.Forward（如果有Forward请求的话）两类数据；Transaction数据中URL点进去的数据就是被访问的具体URL（去掉参数的前缀部分）
3. 请将catFilter存放filter的比较靠前位置，这样可以保证最大可能性监控所有的请求。推荐的是确保MtraceFilter是第一个加载，然后加载CatFilter。MtraceFilter优先于CatFilter


```

<filter>
    <filter-name>cat-filter</filter-name>
    <filter-class>com.dianping.cat.servlet.CatFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>cat-filter</filter-name>
    <url-pattern>/*</url-pattern>
    <dispatcher>REQUEST</dispatcher>
    <dispatcher>FORWARD</dispatcher>
</filter-mapping>
```

filter-mapping可以支持匹配规则，比如如下匹配了/r/开头以及/s/开头的path，这样可以让cat-flter仅仅监控部分URL，而不是所有的URL，不过建议是全量URL。

```
	<filter-mapping>
		<filter-name>cat-filter</filter-name>
		<url-pattern>/r/*</url-pattern>
		<dispatcher>REQUEST</dispatcher>
	</filter-mapping>
	<filter-mapping>
		<filter-name>cat-filter</filter-name>
		<url-pattern>/s/*</url-pattern>
		<dispatcher>REQUEST</dispatcher>
	</filter-mapping>

	
```


filter可以支持过滤掉一些确定的URL，如下配置会过滤掉testPage1;testPage2，如下

```
  <filter>
    <filter-name>cat-filter</filter-name>
    <filter-class>com.dianping.cat.servlet.CatFilter</filter-class>
    <init-param>
            <param-name>exclude</param-name>
            <param-value>testpage1;testpage2</param-value>
    </init-param>
  </filter>

```
    
#### 4.2 RestFull 监控配置
	
- CAT 1.5.4版本之后提供了自动识别url中restful的问题，会自动替换/之间的数字，将数字替换为{num}
- 比如shop/1 shop/2 都会自动归类到 /shop/{num}
- 注意shop/v1 shop/v2 等不会自动归类到/shop/v{num}，cat仅仅替换两个/之间，如果为数字，替换为{num}。对于这类特殊场景，CAT提供了客户端自定义的URL的name功能，只要在HttpServletRequest的设置一个Attribute，就可以解决URL归类到统一的Name
- 在业务运行代码中加入如下代码可以自定义URL下name，这样可以进行自动聚合，代码如下


```	
	HttpServletRequest req = ctx.getHttpServletRequest(); //业务需要在自己上下文获取当前http的请求对象
	req.setAttribute("cat-page-uri", "myPageName");
```

- 在业务运行代码中加入如下代码可以自定义Transaction的Name，比如把URL改为URL.API，比如可以用于排除一些心跳检测url，同时也想在Transaction报表种看到这个心跳的监控

```
	HttpServletRequest req = ctx.getHttpServletRequest();//业务需要在自己上下文获取当前http的请求对象
	req.setAttribute("cat-page-type", "URL.API");
	
```
    

###  5.LOG集成
- cat支持Log4j、Log4j2、Logback的集成。
- cat仅仅关心<font color=#FF4500>log中有exception的日志，并且是error级别的log</font>，也就是error级别的异常监控。其他info，warn级别都不关心，error级别中，如果没有异常堆栈，cat也不关心。
- 在一个messageTree内部，cat内部会针对同样的异常，如果连续调用两次logError(e)，仅仅会上报第一份exception。注意框架打印了一个Exception，如果业务包装此异常为new BizException(e)，这样仍旧算两次异常，上报两份。

   
#### 5.1 Log4j配置
- 业务程序的所有异常都通过记录到CAT中，方便看到业务程序的问题，建议在Root节点中添加此Appendar

- 在Log4j的xml中，加入Cat的Appender
    
```
<appender name="catAppender" class="com.dianping.cat.log4j.CatAppender"></appender>

```

- 在Root的节点中加入catAppender

```
<root>
    <level value="error" />
    <appender-ref ref="catAppender" />
</root>
```
- 注意有一些Log的是不继承root的，需要如下配置

```
<logger name="com.dianping.api.location" additivity="false">
    <level value="INFO"/>
    <appender-ref ref="locationAppender"/>
    <appender-ref ref="catAppender"/>
</logger>
```



