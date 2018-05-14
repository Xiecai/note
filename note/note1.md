###JAVA 项目国际化处理随笔记录  

@(国际化)[java, SpringMVC, IDEA]
 
 **&nbsp;&nbsp;&nbsp;项目最近需要国际化，网上教程有很多也不难，开发过程之中查阅很多资料并总结了一下，希望能帮助你。收集了些配置和设置和一些自己尝试的方法，希望对你有帮助。想写的很细很细，有的可以做参考，也可以忽略，本文的目的就是省去国际化开发的时间。**

项目结构是一个很常见的SpringMVC项目，如图所示
 
 <div align="center"> 
 ![项目结构图](https://raw.githubusercontent.com/Xiecai/note/master/img/note1/projectStructure.png)</div>
   
比较好的方法是基于session的动态切换处理。参考如下配置

----

##配置

#####spring-mvc.xmlz 中加入配置如下：
```xml
    <!-- 国际化资源配置,资源文件绑定器-->
    <bean id="messageSource" class="org.springframework.context.support.ReloadableResourceBundleMessageSource">
        <!-- 国际化资源文件配置,指定properties文件存放位置 -->
        <property name="basename" value="classpath:messages/message" />
        <!-- 如果在国际化资源文件中找不到对应的key，则使用key  -->
        <property name="useCodeAsDefaultMessage" value="true" />
        <!-- 不使用系统默认的Locale配置 -->
        <property name="fallbackToSystemLocale" value="false"/>
    </bean>
    <!-- 动态切换国际化 ,国际化放在session中 -->
    <bean id="localeResolver" class="org.springframework.web.servlet.i18n.SessionLocaleResolver">
        <!-- 默认配置 -->
        <property name="defaultLocale" value="zh_CN" />
    </bean>
    <mvc:interceptors>
        <!-- 国际化操作拦截器 如果采用基于（请求/Session/Cookie）则必需配置 -->
        <bean class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor">
            <!-- 通过这个参数来决定获取那个配置文件 -->
            <property name="paramName" value="language" />
        </bean>
    </mvc:interceptors>
```
一般项目中期才开始考虑国际化，可能像我一样需要处理已在线上的项目 ( *都是被产品经理逼的* )。这里配置 **useCodeAsDefaultMessage**（参考下方代码） 有一个好处，稍后解释，咱们往下看
```xml
    <!-- 如果在国际化资源文件中找不到对应的key，则使用key  -->
        <property name="useCodeAsDefaultMessage" value="true" />
```
发布上线前出现了一个bug，明明在本地可以实现语言切换，但是到仿真环境被默认**卡死**英文了。多方查找网上说原因就是当你本地服务Locale是null或者ResourceBundle没有该Locale的配置文件的话，那么会返回Locale.getDefault()，而仿真系统环境是英文的所以取的en_US。需要如下代码配置处理下，但我在组件localeResolver中配置了defaultLocale并未生效，有点儿不能理解，有知道的请回复我。
```xml
    <!-- 不使用系统默认的Locale配置 -->
        <property name="fallbackToSystemLocale" value="false"/>
```

####下面加入需要语言的配置文件，注意保存文件格式为UTF_8，文件存放在目录的messages下

**message_zh_CN.properties**（中国） & **message_en_US.properties**（英国）[更多](http://www.lingoes.cn/zh/translator/langcode.htm) 这些文件的格式很简单就是 **key = value** 的组合类型于：

```properties
  hello = 你好
```
需要翻译的页面（jsp等）需要引入 spring 标签库
 **<%@taglib prefix="spring" uri="http://www.springframework.org/tags" %>** 需要翻译的内容按如下替换即可。

```html
<spring:message code="hello" />
```
哪儿都能塞，标签里标签外，**javascript**中（js中有个特殊情况，下文详解）。
 <div align="center"> 
 ![素材1](https://raw.githubusercontent.com/Xiecai/note/master/img/note1/material1.png)</div>
需更换哪个语言就在请求路径后追加类似如下形式的后缀。 **key**  就是**spring-mvc.xml**文件 **property name="paramName" value="language"**  配置的value，而 **value** 则是message_**zh_CN**.properties中**zh_CN**的字符串。
- ?language=zh_CN
- &language=en_US

**tips**：切换方案-提供参考(放到head页面)
```html
简易版本 请求中都不会带参数可以这样使用
<a href="?language=zh_CN">中文</a>
<a href="?language=en_US">英文</a>
页面中会有带?key=value的需要自己解决一下
原先带参数再在后缀加?language=zh_CN则不生效，需要加&language=zh_CN
页面地址xxxxxx?language=zh_CN&language=en_US只针对第一个配置有效
请求一次以后，地址中就不需要再携带此参数了，已经由session维护。
<a href="javascript:void(0);" onclick="changeLanguage('language=zh_CN');">中文</a>
```

```javascript
    function changeLanguage(language) {

        var url = window.location.href;

        url = url.split("?language")[0];
        url = url.split("&language")[0];
        
        if (url.indexOf("?") != -1) {
            url += '&'+language;
        } else {
            url += '?'+language;
        }
        
        window.location.href = url;
    }
```

####至此国际化的配置就结束了

---
##处理记录 tips

1.对于开发来说需要维护一个语种一个文件，还记得之前配置中设置的**useCodeAsDefaultMessage**吗？设置它的好处就是，咱们的key可以是**中文**的。对于现在已经在线上发布的项目，起别名的key轻易的替换文字内容可能会引起不必要的bug。设置了useCodeAsDefaultMessage ，形式如：
```html
<spring:message code="你好" />
```
当系统找不到zh_CN配置 **key = 你好** 对应的value时，系统就默认使用key作为value渲染出来。咱只需要把全局页面的中文套上spring的标签即可，方便替换处理。

2.中文作为key对于中文配置**message_zh_CN.properties**来说等于无需配置，但是对于其他语种来说放到**properties**中，中文就是乱码了，那怎么处理？答案是将中文变成 **unicode** 作为 key 放到 properties 中。一段中文转换为一段**unicode**做key/value，前期这么搞还可以，后面越搞越累，越搞越乱，看着unicode一坨一坨的，乱七八糟的也不好维护，那只有把IDEA请出来帮忙了。

#####在project settings - File Encoding，在标红的选项上打上勾
 <div align="center"> 
 ![IDEA设置图](https://raw.githubusercontent.com/Xiecai/note/master/img/note1/IDEAConfiguration.jpg)</div>
#####这样properties中写入中文的key就是中文的显示，然而实际为unciode。

3.key 中不能含 空格,等号,英文冒号 **key会失效**

4.spring 标签可以加入**参数**处理 ，这样对于逻辑拼接的句子可以更好的展示处理 此时需要中文的配置文件兼容一下处理如 ：

```html
<spring:message code="共x条记录" arguments="${pagebean.totalNum}"/>
 多参形式
<spring:message code="x共x条记录" arguments="${name},${pagebean.totalNum}"/>
 可能参数中带 "," 会导致参数引入异常，配置 argumentSeparator 使用
 <spring:message code="晚上date1到date2之间对次日上午date3点前的订单不报价" arguments="
                                      <input value=\"${item.startTime}\" old=\"${item.startTime}\" class=\"aa\" id=\"priceRuleSeting_startTime${vs.index}\"  onkeyup=\"value=this.value.replace(/\D+/g,'')\"/>^
                                      <input value=\"${item.middleTime}\" old=\"${item.middleTime}\" class=\"aa\" id=\"priceRuleSeting_middleTime${vs.index}\"  onkeyup=\"value=this.value.replace(/\D+/g,'')\"/>^
                                      <input value=\"${item.endTime}\" old=\"${item.endTime}\" class=\"aa\" id=\"priceRuleSeting_endTime${vs.index}\"  onkeyup=\"value=this.value.replace(/\D+/g,'')\"/>
                                       " argumentSeparator="^"/>
```

properties 配置如：

```properties 
  共x条记录 = 共{0}条记录
  x共x条记录 = {0}共{1}条记录
  晚上date1到date2之间对次日上午date3点前的订单不报价 = 晚上{0}到{1}之间对次日上午{2}点前的订单不报价
```

5.spring 标签支持 el 语句 但**不支持**三元运算符 如

```html
   ${type == 1 ? <spring:message code="正常"/>:<spring:message code="失败"/>}
```
6.spring 标签的参数没办法接受其他标签 如这样的形式：

```html
<fmt:formatDate value="${order.base.startDatetimeLocal}" pattern="yyyy-MM-dd HH:mm"/>
```
7.js 中不支持带参形式的spring标签 加载顺序导致

8.将key符号带到翻译结果中 如: 司机/车 = driver/car 利于开发

9.将系统要翻译的key全部摘到一个文件中，初期没有翻译时 可参考如下处理，便于测试:

```properties 
  你好=-你好-
```

至此结束，谢谢观看
