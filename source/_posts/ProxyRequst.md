---
title: 使用动态代理对网络请求接口进行面向对象封装
date: 2016-08-19 12:58:03
tags: [Java, Android, 动态代理]
category: Java
---

在android中常规的网络请求方式是把URL地址作为常量放在某个固定的类中，比如定义一个URLConstant，里面定义URL1/URL2/URL3等等，在使用的时候使用URLConstant.URL1这样的形式来获取真实的网络地址。而每一个网络接口的参数列表则是放在另外一个类或者临时赋值，在这样的设计情况下，在网络接口达到一定量的时候会出现一个URL地址找不到对应的参数列表的困扰，因为太多了。
或许在真实实践中可以通过注释或者规范的命名来规避这个问题，但是我们仍然需要考虑有没有一种更好的解决方案呢？比如这样来调用：
```java
//调用：
IPersonServer iPersonServer = ServerBuilder.build(this, IPersonServer.class);
Person person = iPersonServer.getPersonById("003");
//定义：
@Intf("/persons")
public interface IPersonServer {

    @Request("/get")
    Person getPersonById(@Param("personId") String id);
}
```
看起来是不是非常清爽，我们请求一些数据的时候只需要调用相应的java接口以及方法即可，而不用关心这个网络接口需要哪些参数，因为这个java接口已经帮我们约束了，那么接下来我们就记录一下如何实现网络请求的面向对象封装。

<!-- more -->
上面示例中的接口的实际请求地址是http://127.0.0.1:8080/persons/get ，参数列表为personId=003
要达到这个目的我们需要解析接口注解中的的参数，我们首先为interface、function、param分别定义注解
接口注解：
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Intf {
    String value();
}
```
方法注解：
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Request {
    String value() default "";
}
```
参数注解：
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface Param {
    String value();
}
```
这时候我们就可以定义实例中的那种形式的接口了，但是我们要如何实现java接口和网络接口的转换呢？
动态代理是个非常不错的方法，直接上代码：

```java
public class ServerBuilder {
    private static final String URLHost = "http://127.0.0.1:8080";

    public static <T> T build(Context context, final Class<T> serverCls) {
        Object proxyInstance = Proxy.newProxyInstance(context.getClassLoader(), new Class[]{serverCls}, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                String URL = URLHost;
                Intf annoIntf = serverCls.getAnnotation(Intf.class);
                URL += annoIntf.value();
                Request annoRequest = method.getAnnotation(Request.class);
                URL += annoRequest.value();
                Annotation[][] parameterAnnotations = method.getParameterAnnotations();
                Map<String, Object> paramMap = new HashMap<>();
                for (int i = 0; i < parameterAnnotations.length; i++) {
                    Annotation[] parameterAnnotation = parameterAnnotations[i];
                    for (Annotation annoParam : parameterAnnotation) {
                        if (annoParam instanceof Param) {
                            String key = ((Param) annoParam).value();
                            paramMap.put(key, args[i]);
                        }
                    }
                }
                Log.i("ProxyTest", "网络请求地址：" + URL);
                Log.i("ProxyTest", "网络请求参数：" + paramMap);
                // TODO: 网络请求，并将返回值反序列化为object返回
                //模拟返回值对象
                Person person = new Person();
                person.id = "003";
                person.name = "张三";
                return person;
            }
        });
        return (T) proxyInstance;
    }
}
```
为了让流程看起来更加请清晰，上列代码中暂时把Annotation的非空判断去掉了。
这个方法中传入了context和接口的Class对象，返回了一个接口的动态代理对象。
首先根据传入方法的Class对象获取接口的注解中所定义的url路径，当代理对象的方法被调用时，Method对象会被传入invoke回调中，接着获取method的注解中所定义的url，接着把这两个url和Host地址进行拼接即可得到我们需要的真实的URL
在代理回调方法中的第三个参数是方法的参数列表，这是一个Object数组，并不会把参数的定义名称传进来，在对代码进行混淆之后的class文件中也并不会存储参数名，所以这里们定义了Param注解来指定Url需要参数名，Method的getParameterAnnotations方法会Annotation的二纬数组，这个二纬数组描述了方法的**全部**参数上所定义的**全部**注解，而这个有序的数组是和参数列表是一一对应的。那么我们就可用从这个参数注解的二维数组中拿到我们想要的数据了，把从注解中拿到的参数的名称和值放入一个hashmap中以供网络请求使用。

鉴于本篇只是描述如何使用动态代理封装URL，网络请求的实现和返回值的反序列化就不在这里赘述，这里使用一个Person对象来模拟返回值；

运行结果如下：
```
I/ProxyTest: 网络请求地址：http://127.0.0.1:8080/persons/get
I/ProxyTest: 网络请求参数：{personId=003}
I/ProxyTest: 获得动态代理接口返回值：Person对象：{id:003, name:张三}
```

这样的实现方式在互联网上也是存在的，比如现在比较火的Retrofit，和RxJava一起使用效果更佳。

