---
layout: default
title: 基于spring-kafka实现多环境消息隔离
---
笔者接到一个需求，背景大概是这样的：一个服务需要部署多套环境，多套环境共用一个Kafka集群，为了保证一个环境的消息不被另外一个环境消费，需要实现环境之间的消息隔离。正常而言，不同环境使用不同的topic是最简单有效的方式。然而笔者和公司的SRE沟通，kafka集群为多个部门共用，kafka集群的topic数量有限制，SRE不允许业务方自己创建topic。因此通过topic来隔离的方案不可行，需要寻找其他思路解决。  

笔者在方案设计之初，重点考虑两点：1、对业务的侵入性要尽可能的轻，最好是不改; 2、kafka生产和消费的方式很多，可以单个消费，可以批量消费，可以手动提交，也可以自动提交，因此方案覆盖的场景要尽可能地全面。笔者之前对kafka的理解更多停留在理论上，对kafka的使用也更多是基于封装好的框架，比如当前公司使用的spring-kafka。为了较好地解决需求，笔者比较深入地了解spring-kafka，充分利用sping-kafka提供的扩展来实现。回顾整个开发过程，笔者发现自己对kafka的认识，以及对spring-kafka框架的理解都有很大的缺陷，因此用文章整理记录下。  

既然不能新建topic，那就意味着多个环境的消息在同一个topic中，也意味着必须有标识来区分不同环境的消息。不同环境的消费者，应当属于不同的消费组，否则就会发生不同环境消费了彼此消息的问题，因此同样的代码，部署在不同的环境，应该注册不同的消费组。传递给消费者的注册函数的消息，必须把不属于相应环境的信息过滤掉。总的来说，方案设计的重点有三：1、不同环境生产的消息具有不同的标识；2、不同环境的消费者属于不同的消费组，消费组应该自适应；3、不同环境的消费者获取的消息，是经过过滤的。笔者先介绍方案设计实现，然后介绍spring-kafka的整体设计实现，以及对kafka的一些认识误区。  

方案设计实现
====

消息标识
----

一个spring-kafka 的消息，包括消息体，和消息头Headers(Header列表)。消息结构如下：
```
public class ProducerRecord<K, V> {
    private final String topic;
    private final Integer partition;
    private final Headers headers;
    private final K key;
    private final V value;
    private final Long timestamp;
}
```
可以通过设置不同的Header，来进行消息标识。spring-kafka发送消息的类是kafkaTemplate。观察KafkaTemplate的发送方法，主要如下所示：

```
    public ListenableFuture<SendResult<K, V>> send(String topic, @Nullable V data) {
        ProducerRecord<K, V> producerRecord = new ProducerRecord(topic, data);
        return this.doSend(producerRecord);
    }

    public ListenableFuture<SendResult<K, V>> send(String topic, K key, 
            @Nullable V data) {
        ProducerRecord<K, V> producerRecord = new ProducerRecord(topic, key, data);
        return this.doSend(producerRecord);
    }

    public ListenableFuture<SendResult<K, V>> send(String topic, Integer partition, 
            K key, @Nullable V data) {
        ProducerRecord<K, V> producerRecord = new ProducerRecord(topic, partition, 
            key, data);
        return this.doSend(producerRecord);
    }

    public ListenableFuture<SendResult<K, V>> send(String topic, Integer partition, 
            Long timestamp, K key, @Nullable V data) {
        ProducerRecord<K, V> producerRecord = new ProducerRecord(topic, partition, 
            timestamp, key, data);
        return this.doSend(producerRecord);
    }

    public ListenableFuture<SendResult<K, V>> send(ProducerRecord<K, V> record) {
        Assert.notNull(record, "'record' cannot be null");
        return this.doSend(record);
    }

   protected ListenableFuture<SendResult<K, V>> doSend(ProducerRecord<K, V> producerRecord) {}
```
可以发现有多个重载的send方法，这些方法的逻辑都是将消息体封装成ProducerRecord，然后调用doSend进行发送。笔者首先想到用Aop，在调用doSend方法时插入一段逻辑，这个逻辑就是将环境信息放入到Header中。但是doSend 属于方法内调用，无法使用Aop。既然无法使用动态代理，那是否可以使用静态代理呢，即创建自定义的KafkaTemplat，调用父类的doSend之前先执行插入Header逻辑。要想实现这个思路，首先得弄清楚spring-kafka是如何引入KafkaTemplate实例的，答案在KafkaAutoConfiguration 类中，如下所示：

```
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnClass({KafkaTemplate.class})
@EnableConfigurationProperties({KafkaProperties.class})
@Import({KafkaAnnotationDrivenConfiguration.class, 
    KafkaStreamsAnnotationDrivenConfiguration.class})
public class KafkaAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean({KafkaTemplate.class})
    public KafkaTemplate<?, ?> kafkaTemplate(ProducerFactory<Object, Object> kafkaProducerFactory, 
        ProducerListener<Object, Object> kafkaProducerListener, 
        ObjectProvider<RecordMessageConverter> messageConverter) {
        KafkaTemplate<Object, Object> kafkaTemplate = new KafkaTemplate<>(kafkaProducerFactory); 
 messageConverter.ifUnique(kafkaTemplate::setMessageConverter); 
 kafkaTemplate.setProducerListener(kafkaProducerListener); 
 kafkaTemplate.setDefaultTopic(this.properties.getTemplate().getDefaultTopic()); 
        return kafkaTemplate;
    }
}
```

可见在KafkaAutoConfiguration中，如果Bean容器中没有KafkaTemplate类型的Bean，便会创建一个KafkaTemplate。那spring-kafka又是如何引入KafkaAutoConfiguration的？答案是在spring-kafka 包的spring.factories文件中，这也是spring-boot支持扩展的核心方式之一。因此只需定义自己的KafkaTemplate，便会自动取代原生的KafkaTemplate，如下所示:  

```
public class GlspKafkaTemplate<K, V> extends KafkaTemplate<K, V> {
    @Override
   protected ListenableFuture<SendResult<K, V>> doSend(final ProducerRecord<K, V> producerRecord) {
        /**
        ** 在producerRecord的HEADER中添加环境信息
        */
        return super.doSend(producerRecord);
    }
}
```

消费组自适应
----
在 spring-kafka 中，消费者是用注解KafkaListener来注册的。消费者属于哪个消费组，消费哪个topic的消息，是在注解KafkaListener中的topics和groupId 属性中定义。如果修改每个KafkaListener 注解group属性，添加环境信息，业务侵入性太大，严重违背了初衷。于是笔者探究 spring-kafka 是如何解析注解KafkaListener的。在KafkaListener类的注释中就提到，解析KafkaListener的操作是在KafkaListenerAnnotationBeanPostProcessor中完成。KafkaListenerAnnotationBeanPostProcessor 实现了BeanPostProcessor 接口，每个Bean初始化前后都会调用相应的方法，KafkaListenerAnnotationBeanPostProcessor 解析 KafkaListener 注解的思路就是当Bean初始化完毕回调方法时，判断Bean类或者方法上是否有KafkaListener注解，如果有的话就会执行相应解析操作。笔者仔细阅读了KafkaListenerAnnotationBeanPostProcessor类的代码，发现在解析过程中给开发者预留了扩展点——AnnotationEnhancer，spring-kafka会将解析结果作为参数回调接口，允许开发者对解析结果进行修改调整。接口定义如下:  

```
public interface AnnotationEnhancer extends BiFunction<Map<String, Object>, 
    AnnotatedElement, Map<String, Object>> {

}
```

这个接口并没有方法，只是继承了BiFunction并定义了泛型。笔者正是使用这个接口，修改KafkaListener的groupId属性，在开发者配置的groupId的基础上，在加一个环境变量的$表达式，在对开发者透明的情况下，将环境信息写入groupId属性，实现不同环境的消费者，注册属于不同的消费组，实现代码如下:  

```
@Component
public class DefaultAnnotationEnhancer implements 
    KafkaListenerAnnotationBeanPostProcessor.AnnotationEnhancer {

 @Override
 public Map<String, Object> apply(Map<String, Object> stringObjectMap, 
    AnnotatedElement annotatedElement) {
        if (stringObjectMap.containsKey("groupId")) {
            String groupId = stringObjectMap.get("groupId").toString();
            String newGroupId = groupId + "-${" + Constants.ENV_NAME + "}";

            log.debug("enhance groupId, before = {}, after = {}", groupId, newGroupId);
            stringObjectMap.put("groupId", newGroupId);
        }
        return stringObjectMap;
    }
 }
```

消息过滤
----
不同环境的消费者属于不同的消费组，那么每个环境的消费者会收到topic中全量的消息，因此有必要将不属于本环境的消息过滤掉。那应该如何实现过滤呢？spring-kafka 在回调开发者注册的处理函数时，也预留了扩展接口：RecordInterceptor 和 BatchInterceptor。RecordInterceptor是消息单个处理时回调，BatchInterceptor是消息批量处理时回调。其中最主要的方法如下:  

```
default ConsumerRecord<K, V> intercept(ConsumerRecord<K, V> record, 
    @SuppressWarnings("unused") Consumer<K, V> consumer) {}

ConsumerRecords<K, V> intercept(ConsumerRecords<K, V> records, Consumer<K, V> consumer)
```
RecordInterceptor 和 BatchInterceptor 不仅提供了消息，还提供的Consumer。这样如果发现不是本环境的消息，可以使用consumer提交offset，使消费者可以继续向前消费。具体完整代码，见[github](https://github.com/fsxtiger/code/tree/master/kafka-isolute/src/main/java/kafka)。

总体方案就是如此，可以覆盖绝大多数场景。而对于业务开发者，也只是增加一行配置就行，侵入性非常低。实践下来，笔者觉得无论是spring-kafka的框架使用，还是对kafka的理解，都太粗糙了。正好趁此机会好好补充一下。在介绍spring-kafka的整体设计之前，笔者预留几个问题。这几个问题，笔者之前都是毫不犹豫给出轻率答复，但其实理解都是错误的。  

1. 如果消息被注册函数正常处理，但是忘记提交offset，是会一直block在这个消息上，还是会继续消费处理下一个消息
2. 如果消息处理过程中，抛出一个未被捕获的异常，是会一直block在这个消息上，还是会继续消费处理下一个消息
3. 如何有效的实现消息的自动重试

spring-kafka整体设计
====
spring-kafka框架的核心是：KafkaListenerAnnotationBeanPostProcessor 和 MessageListenerContainer，大部分逻辑都在其中。spring-kafka 整体设计结构如下:  
<img src="http://dbp-resource.cdn.bcebos.com/9abe0e71-8c18-ab1c-22ac-a4f210feae48/spring-kafka.jpg" height = "500" width="100%"/>

其中核心类是KafkaListenerAnnotationBeanPostProcessor 和 MessageListenerContainer 类。KafkaListenerAnnotationBeanPostProcessor 扫描、注册以及启动消费者，MessageListenerContainer 则负责具体的消息拉取，以及回调消息处理函数。当然笔者介绍的只是冰山一角，spring-kafka的东西远不止这些，笔者也只能尽其所能才能读懂一些皮毛而已。  

KafkaListenerAnnotationBeanPostProcessor 关键是实现了BeanPostProcessor、InitializingBean以及SmartInitializingSingleton接口，可以在spring-kafka不同的生命阶段，收到不同的回调。实现了InitializingBean的afterPropertiesSet方法，可以在Bean实例化完毕属性设置完成时调用，KafkaListenerAnnotationBeanPostProcessor在这个阶段创建了注解增强属性enhancer，后续对@KafkaListener注解的增强就是通过这个方法实现。postProcessAfterInitialization方法是在Bean初始化之后调用，此时可以查看Bean的class或者method是否有相应的注解，如果有的话就执行相应的解析动作，并调用enhancer对行为进行增强。afterSingletonsInstantiated是在所有单例Bean实例化完之后调用，此时所有的注解解析和增强动作都已经完成了，在这个方法中启动所有注册的消费者。  

MessageListenerContainer 是一个监听器容器，根据不同的配置创建不同的容器实现。容器中的逻辑也是类似，不断使用KafkaConsumer从Kafka中拉取消息，然后回调监听器方法。SeekToCurrentErrorHandler则是针对消息处理失败场景，spring-kafka提供的一个兜底逻辑。最后笔者给出预留问题的答案:  

1. 会继续消费下一条，表现和正常提交了offset 一样。但是由于实际上并没有提交，消息都堆积在kafka队列中，如果一旦发生rebalance，这些消息都会重新消费。这是新版kafka的规范决定的，消息拉取和offset提交是两个独立的过程，只要没有发生rebalance, 这个过程会持续下去。
2. 这个需要看 KafkaListener注解是否配置了errorHandler，如果配置了并且被正常处理，这条消息就直接过了。如果没有配置，由于Spring-kafka默认的消息处理方式是SeekToCurrentErrorHandler，默认处理10次，如果10次全部失败还是会过的。如果自定义的异常处理器会抛出的异常，又会回到SeekToCurrentErrorHandler逻辑，避免程序陷入不断重试的死循环之中。
3. 使用SeekToCurrentErrorHandler 可以很优雅地解决问题。

spring-kafka的知识还有很多，比如ConsumerCoordinator这些，笔者尚未深入了解。通过本文，笔者分享了一些之前对Kafka的误区，然后利用spring-kafka提供的一些扩展接口实现了一个消息隔离的小功能。