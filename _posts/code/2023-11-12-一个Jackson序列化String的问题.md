---
layout: default
title: 一个Jackson序列化String的问题
---
笔者收到一个同事的报错，背景大概是这样：通过spring-data-redis提供的RedisTemplate类操作redis，get操作的时候抛类型转换异常，提示无法将java.lang.Integer类型转换为java.lang.String。首先声明spring-data-redis用的是2.5.11版本。具体代码如下：
```
@Configuration
public class GlspRedisTemplateUtil {

    @Resource
    private RedisTemplate<String, String> redisTemplate;
    
    public String get(String key) {
        return redisTemplate.opsForValue().get(key);
    }

}
```
同事对比代码发现，get方法逻辑有被改动过，  之前是String.valueOf( redisTemplate.opsForValue().get(key))。同事打算恢复代码来解决问题。但笔者记得redis中并没有int类型，所有的基本类型都是以string形式存储的，为什么出现Integer类型，以至于发生转换错误，因此决定一探究竟。  

笔者调试发现，RedisTemplate读取的结果类型是byte[]，只有一个元素:51，确认是对应value的ASCII。经过Jackson反序列化之后，结果类型是java.lang.Integer。如下所示：  

<img src="http://dbp-resource.cdn.bcebos.com/9abe0e71-8c18-ab1c-22ac-a4f210feae48/fastJson1.jpg" height = "300" width="100%"/>

同时笔者也验证，如果传入的Type是String.class，则解析的结果类型为String。说明Jackson针对数字字符串，默认转换成整型，如果明确指定type为String.class，也可以正常转换为String类型。因此get方法抛出异常的直接原因，是因为存入的值是数字字符串，在get的时候由于泛型擦除，结果被解析成了Integer，最终导致类型转换异常。笔者在redis命令行直接使用set命令，也可以复现异常。  

笔者不仅疑惑，代码中调用set，如果设置为数字字符串，get的时候难道都会抛类型转换异常。如果说RedisTemplate 不支持 set 数字字符串，显然是严重bug。笔者继续调试set方法，如下所示:  

<img src="http://dbp-resource.cdn.bcebos.com/9abe0e71-8c18-ab1c-22ac-a4f210feae48/fastJson2.jpeg" height = "300" width="100%"/>

笔者发现数字字符串"6"，Jackson解析后的字节数组为[34, 54, 34]，ASCII码34 对应的字符是"，因此在序列化时，Jackson将字符串的引号也进行了序列化。笔者写Demo验证，如果序列化结果中带上”, 即便type是Object.class，Jackson 会转换成String。通过命令行直接查看redis也可以证明：  

<img src="http://dbp-resource.cdn.bcebos.com/9abe0e71-8c18-ab1c-22ac-a4f210feae48/FastJson4.jpeg" height = "300" width="100%"/>

既然set操作Jackson 会自动将" 也一并序列化，那同事是如何在redis 中存入数字字符串的，经过沟通同事是通过incr方法来设置的。incre命令的value最终值是由redis计算得出的，结果是不带引号的，所以get的时候会抛异常。同时笔者也验证了，如果对一个key首先进行set，然后incr，同样会抛异常，而且这种情况不仅使用Jackson会出现，使用FastJson也会出现。  

那么应该如何设计这段代码，可以避免get时抛出异常，同时incr时也可以正常使用。笔者根据经验，觉得较好的设计可以有如下两种：
1. 当前代码不做修改，get只允许set方式设置的值，incr设置的值依旧不能get。incr方法已经将value返回，很容易推导出incr之前的值来进行逻辑判断，没有必要单独get
2. 重新定义泛型类型，这样既可以设置String，也可以Integer，get的时候默认为String，也可以转换成所需要的
```
@Configuration
public class GlspRedisTemplateUtil {
    @Resource
    private RedisTemplate<String, Serializable> redisTemplate;

    public void  set(String key, Serializable value) {
        redisTemplate.opsForValue().set(key, value);
    }

    public String get(String key) {
        return String.valueOf(redisTemplate.opsForValue().get(key));
    }

    public <T> T get(String key, Class<T> clazz) {
        return  clazz.cast(redisTemplate.opsForValue().get(key));
    }
}
```