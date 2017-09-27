---
title: SpringMVC自定义属性编辑器
date: 2017-08-15 14:51:42
tags: [springmvc,自定义属性编辑器]
categories: spring
---
# 参数绑定流程
参数绑定：把请求中的数据，转化成指定类型的对象，交给处理请求的方法
 ![](http://i.imgur.com/Pw0rmxB.jpg)
1. 请求进入到DisptacherServlet，卸下请求中的数据
2. DisptacherServlet将请求中的数据发送给Controller
3. 获取Controller需要接收的参数类型，将参数类型和请求数据发送给DataBinder
4. DataBinder将参数类型和请求数据再发给TypeConverter，由TypeConverter装配成一个bean
5. TypeConverter根据bean中的成员类型，在PropertyEditorRegistry中查找已注册的PropertyEditor
6. PropertyEditor将数据setter进bean中的成员
7. TypeConverter将装配好的bean返回给DataBinder
8. DataBinder将装配bean交给处理请求的方法
在参数绑定的过程TypeConverter和PropertyEditor是最核心的数据转化成对象（非序列化）的过程。TypeConverter负责将数据转化成一个bean。PropertyEditor负责将数据转化成一个成员字段
<!--more-->

# 属性编辑器
PropertiesEditor负责转化简单对象，因为http请求都是以字符串的形式，所以一般都是根据String来转换springmvc提供了很多默认的属性编辑器，在org.springframework.beans.propertyeditors包中，比如

> CustomBooleanEditor.class，String 转换 Boolean
> CustomCollectionEditor.class，String 转换 Collection
> CustomDateEditor.class，String 转换 Date
> CustomMapEditor.class，String 转换 Map
> CustomNumberEditor.class，String 转换  int、floot、double..

所有的属性编辑器都是继承PropertiesEditorSupport接口，默认的属性编辑器，Spring在启动的时候会自动加载。
除此之外，如果要装配的属性没有合适的编辑器，还可以自定义属性编辑器注册了自定义的属性编辑器之后，在CustomEditorConfigurer中注册，应用全局都可以使用这个属性编辑器，因为属性编辑器的工厂是全局作用域的

# 自定义springMVC的属性编辑器主要有两种方式，
1.	一种是使用@InitBinder标签在运行期注册一个属性编辑器，这种编辑器只在当前Controller里面有效；
2.	还有一种是实现自己的WebBindingInitializer，然后定义一个AnnotationMethodHandlerAdapter的bean，在此bean里面进行注册，这种属性编辑器是全局的。
## 第一种方式：

```java
@Controller  
public class GlobalController {       
    @RequestMapping("test/{date}")  
    public void test(@PathVariable Date date, HttpServletResponse response) throws IOException  
        response.getWriter().write( date);  
  
    }  
      
    @InitBinder//必须有一个参数WebDataBinder  
    public void initBinder(WebDataBinder binder) {  
        binder.registerCustomEditor(Date.class, new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"), false));  
  
                binder.registerCustomEditor(int.class, new PropertyEditorSupport() {  
  
            @Override  
            public String getAsText() {  
                // TODO Auto-generated method stub  
                return getValue().toString();  
            }  
  
            @Override  
            public void setAsText(String text) throws IllegalArgumentException {  
                // TODO Auto-generated method stub  
                System.out.println(text + "...........................................");  
                setValue(Integer.parseInt(text));  
            }  
              
        });  
    }     
}  
```
  这种方式这样写了就可以了
## 第二种：
1.定义自己的WebBindingInitializer

```java
import java.util.Date;  
import java.text.SimpleDateFormat;  
  
import org.springframework.beans.propertyeditors.CustomDateEditor;  
import org.springframework.web.bind.WebDataBinder;  
import org.springframework.web.bind.support.WebBindingInitializer;  
import org.springframework.web.context.request.WebRequest;  
  
public class MyWebBindingInitializer implements WebBindingInitializer {  
  
    @Override  
    public void initBinder(WebDataBinder binder, WebRequest request) {  
        // TODO Auto-generated method stub  
        binder.registerCustomEditor(Date.class, new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"), false));  
    }  
} 
``` 
2.在springMVC的配置文件里面定义一个AnnotationMethodHandlerAdapter，并设置其WebBindingInitializer属性为我们自己定义的WebBindingInitializer对象

```xml
<bean class="org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter">    
	        <property name="cacheSeconds" value="0"/>    
	        <property name="webBindingInitializer">    
	            <bean class="com.xxx.blog.util.MyWebBindingInitializer"/>    
	        </property>    
	    </bean>
```    
 第二种方式经过上面两步就可以定义一个全局的属性编辑器了。
注意：当使用了<mvc:annotation-driven />的时候，它 会自动注册DefaultAnnotationHandlerMapping与AnnotationMethodHandlerAdapter 两个bean。这时候第二种方式指定的全局属性编辑器就不会起作用了，解决办法就是手动的添加上述bean，并把它们加在<mvc:annotation-driven/>的前面。如果不生效，则将手动注册AnnotationMethodHandlerAdapter改为手动
参考：http://www.cnblogs.com/wewill/p/5676920.html
     http://jiji87432.iteye.com/blog/1781857
