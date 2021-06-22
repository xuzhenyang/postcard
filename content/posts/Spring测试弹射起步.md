---
title: "一招让 Spring 本地测试弹射起步"
date: 2021-06-22
tags: ["技术博客"]
draft: false
keywords: ["lilpilot", "Spring", "Spring启动", "BeanFactoryPostProcessor", "Spring本地测试"]
---

开发过程中，在本地启动系统来进行一些测试、联调是很日常的操作。然而随着项目不断壮大，系统内会依赖越来越多的中间件或服务，例如 Rocket MQ、Dubbo、定时任务等。在系统启动的时候，需要对这些组件进行初始化，导致启动过程缓慢无比。

系统启动 5 分钟，测试运行 3 秒钟，这谁顶得住啊。而且在本地测试时，通常也用不到这些组件。

如果你也有同样的困扰，接下来就教你一招，利用 Spring 的扩展点 BeanFactoryPostProcessor，无侵入地“阉割”这些拖慢启动时间的组件，让你的系统一键弹射起步。

我们就以 Rocket MQ 为例，在启动的时候加载 5 个消费者。

```java
public class RocketMQConfiguration {

    /**
     * SpringBoot 启动时加载所有消费者
     */
    @PostConstruct
    public void initConsumer() {
        Map<String, AbstractRocketConsumer> consumers = applicationContext.getBeansOfType(AbstractRocketConsumer.class);
        if (CollectionUtils.isEmpty(consumers)) {
            log.info("no consumers");
        }
        for (String beanName : consumers.keySet()) {
            AbstractRocketConsumer consumer = consumers.get(beanName);
            consumer.init();
            createConsumer(consumer);
            log.info("init consumer: title {} , topics {} , tags {}", consumer.consumerTitle,
                    consumer.topics, consumer.tags);
        }
    }
    
}
```

运行一下测试类，系统启动耗时 30 秒，其中 Bean 的初始化过程就占了 28 秒。

![](https://p5.toutiaoimg.com/origin/pgc-image/08a7a4e0511e4ba4873dc39e199f42e4.jpg)

这里计算 Bean 初始化耗时的方式，是使用了 BeanPostProcessor 的前后扩展点，收集所有 Bean 在初始化前后的时间点，用最大时间减去最小时间，来计算整个过程的耗时。

接下来我们利用 BeanFactoryPostProcessor 这个扩展点，缩短初始化的时间。

首先回顾一下 Spring 初始化 Bean 的过程，大体上可以划分为解析配置，初始化 BeanFactory，注册 BeanDefinition，注册 BeanPostProcessor，然后初始化所有的单例 Bean，最后执行 BeanPostProcessor 的回调方法。

![Bean 的初始化过程](https://p9.toutiaoimg.com/origin/pgc-image/005d789c00954c57aa07a135b537a162.png)

如果你熟悉 Spring 的源码，这段逻辑就在 org.springframework.context.support.AbstractApplicationContext#refresh 里，细看一下这段过程，其中会执行 invokeBeanFactoryPostProcessors(beanFactory) 方法，而 BeanFactory 里保存了所有 bean 的定义，也就是 BeanDefinition。之后，Spring 就是根据这些 BeanDefinition 来初始化对应的 bean。

![](https://p9.toutiaoimg.com/origin/pgc-image/af9cbfdf4a1b4ecbbc6977ed70536ff9.jpg)

因此，我们就可以自定义一个 BeanFactoryPostProcessor ，也就是在 bean 进行初始化之前，使用一招“狸猫换太子”，悄悄删除一些不需要的 BeanDefinition，这样，Spring 就不会初始化这些 bean 了。你也可以根据需要和实际情况，做一些针对性的处理。

```java
public class SpringTestLaunchControl implements BeanFactoryPostProcessor {

    private static final Set<String> EXCLUDE_BEAN_CLASSES = Sets.newHashSet(
            "co.lilpilot.springtestfield.mq.Consumer1",
            "co.lilpilot.springtestfield.mq.Consumer2",
            "co.lilpilot.springtestfield.mq.Consumer3",
            "co.lilpilot.springtestfield.mq.Consumer4",
            "co.lilpilot.springtestfield.mq.Consumer5"
            );

    private static final Set<String> ANNOTATION_BEAN_CLASSES = Sets.newHashSet();

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        DefaultListableBeanFactory defaultBeanFactory = (DefaultListableBeanFactory) beanFactory;

        ArrayList<String> definitionNames = Lists.newArrayList(defaultBeanFactory.getBeanDefinitionNames());

        for (String definitionName : definitionNames) {
            BeanDefinition beanDefinition = defaultBeanFactory.getBeanDefinition(definitionName);
            String beanClassName = beanDefinition.getBeanClassName();
            // 根据类名移除 BeanDefinition
            if (EXCLUDE_BEAN_CLASSES.contains(beanClassName)) {
                defaultBeanFactory.removeBeanDefinition(definitionName);
            }
            // 根据注解移除 BeanDefinition
            if (beanDefinition instanceof ScannedGenericBeanDefinition) {
                ScannedGenericBeanDefinition scannedGenericBeanDefinition = (ScannedGenericBeanDefinition) beanDefinition;
                AnnotationMetadata metadata = scannedGenericBeanDefinition.getMetadata();
                for (String annotationBeanClass : ANNOTATION_BEAN_CLASSES) {
                    if (metadata.hasAnnotation(annotationBeanClass)) {
                        defaultBeanFactory.removeBeanDefinition(definitionName);
                    }
                }
            }
        }
    }

}
```

在 BeanFactoryPostProcessor 中，获得 BeanFactory，然后自定义一些匹配规则，移除掉 factory 里的 BeanDefinition。

最后我们在测试类中配置上这个 processor，仅针对测试时才有效，不会影响系统正常启动。

```java
@SpringBootTest
class SpringTestFieldApplicationTests {

    @TestConfiguration
    static class MyConfig {
        @Bean
        public BeanFactoryPostProcessor beanFactoryPostProcessor() {
            return new SpringTestLaunchControl();
        }
    }

}

```

来重新运行一下测试，看看效果。

![](https://p9.toutiaoimg.com/origin/pgc-image/a1aed1f6d1714ec2aeeaa7195496d875.jpg)

好家伙，系统启动仅用时 5 秒，Bean 的初始化缩减到了 3 秒（说明 MQ 消费者的初始化是真滴慢），弹射起步！

总结一下，本文中，利用 BeanFactoryPostProcessor 这个扩展点，移除了测试过程中所不需要的一些 bean，从而缩短了系统启动时间，而且仅针对测试启动环境有效，对系统正常启动无侵入。

希望有帮助到你。

