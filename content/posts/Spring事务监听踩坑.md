---
title: "Spring 事务事件监听，为什么会出现事务失效？"
date: 2021-03-20
tags: ["技术博客"]
draft: false
keywords: ["lilpilot", "Spring", "Spring事务", "Spring事务事件监听", "Spring事务失效"]
---

Spring 在 4.2 版本之后提供了@TransactionlEventListener 注解，可以很方便地在事务提交后做一些处理，但是如果使用不当，或者没有正确理解其背后的运行逻辑，很容易踩坑甚至导致线上故障。

之前工作中就遇到了一个问题，在事务监听时，做了一些事务操作，但是这个事务并没有生效。

今天我们就来深入了解一下，这个问题是怎么产生的，又该如何解决。

## 问题复现

我们来模拟一个很简单的场景：创建订单的时候会发布“订单已注册”的事件，在事件监听里保存操作记录，再发布“操作记录已保存”的事件，最后在这个事件监听里做一些逻辑。

以下代码中省略了一些不重要的实现。

首先是 OrderService，createOrder() 方法里保存订单记录，发布“订单已注册”的事件：

```
public class OrderService {

    @Transactional
    public void createOrder() {
        String orderNo = "test_no";
        Order order = new Order(orderNo);
        orderRepository.save(order);
        log.info("publish OrderCreatedEvent");
        applicationContext.publishEvent(new OrderCreatedEvent(orderNo));
    }

}

```

“订单已注册”的事件监听里，调用 operationService.saveOperation()：

```
public class OrderCreatedEventListener {

    @TransactionalEventListener
    public void handle(OrderCreatedEvent event) {
        log.info("handle OrderCreatedEvent : " + event.getOrderNo());
        operationService.saveOperation(event.getOrderNo(), "创建订单");
    }

}

```

OperationService.saveOperation()，保存操作记录，并发布“操作记录已保存”的事件：

```
public class OperationService {

    @Transactional
    public void saveOperation(String orderNo, String info) {
        Operation operation = new Operation(orderNo, info);
        operationRepository.save(operation);
        log.info("publish OperationSavedEvent");
        applicationContext.publishEvent(new OperationSavedEvent(orderNo, info));
    }

}

```

“操作记录已保存”的事件监听里，打印一下日志，代替后续操作：

```
public class OperationSavedEventListener {

    @TransactionalEventListener
    public void handle(OperationSavedEvent event) {
        log.info("handle OperationSavedEvent : " + event.getOrderNo());
    }

}

```

开始测试，调用一下 orderService.createOrder() 方法，看一下日志打印：

```
Hibernate: insert into order_entity (id, order_no) values (null, ?)
INFO c.l.s.service.OrderService    : publish OrderCreatedEvent
INFO c.l.s.event.OrderCreatedEventListener    : handle OrderCreatedEvent : test_no
INFO c.l.s.service.OperationService: publish OperationSavedEvent

```

奇怪的事情发生了！数据库里只写入了订单数据，并没有写入操作记录，而且发布了 OperationSavedEvent 事件后，监听回调没有执行。

## 问题排查

先翻阅一下官方文档，在 [事务事件](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction-event) 章节内，有这么一段提示：

![](https://youjb.com/images/2021/03/20/spring-777ea611e5697038.jpg)

最后一句话的意思是：在事务事件监听内，已经没有可供加入的事务。

回顾一下上面的问题代码，OrderService.createOrder() 是一个事务方法，这个事务提交后，触发了 OperationSavedEventListener，而在这个监听方法里，OperationService.saveOperation() 也是一个事务方法，传播类型为默认，即会加入当前事务。

但是在执行 saveOperation() 时，前面的事务已经完成了提交，所以没办法加入，导致操作记录保的事务没有真正执行。又因为操作记录保存的事务没有执行，所以没有触发 OperationSavedEventListener。

哦~大概明白了问题所在，我们进入 Spring 源码看一看是不是真的如此。

首先将 JPA 的日志级别调整为 debug

```
logging.level.org.springframework.orm.jpa=debug
```

再运行一下，看看日志：

```
DEBUG o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [co.lilpilot.springtestfield.service.OrderService.createOrder]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
DEBUG o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(1115296438<open>)] for JPA transaction
DEBUG o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@fe87ddd]
DEBUG o.s.orm.jpa.JpaTransactionManager        : Found thread-bound EntityManager [SessionImpl(1115296438<open>)] for JPA transaction
DEBUG o.s.orm.jpa.JpaTransactionManager        : Participating in existing transaction
Hibernate: insert into order_entity (id, order_no) values (null, ?)
INFO c.l.s.service.OrderService    : publish OrderCreatedEvent
DEBUG o.s.orm.jpa.JpaTransactionManager        : Initiating transaction commit
DEBUG o.s.orm.jpa.JpaTransactionManager        : Committing JPA transaction on EntityManager [SessionImpl(1115296438<open>)]
INFO c.l.s.event.OrderCreatedEventListener    : handle OrderCreatedEvent : test_no
DEBUG o.s.orm.jpa.JpaTransactionManager        : Found thread-bound EntityManager [SessionImpl(1115296438<open>)] for JPA transaction
DEBUG o.s.orm.jpa.JpaTransactionManager        : Participating in existing transaction
DEBUG o.s.orm.jpa.JpaTransactionManager        : Found thread-bound EntityManager [SessionImpl(1115296438<open>)] for JPA transaction
DEBUG o.s.orm.jpa.JpaTransactionManager        : Participating in existing transaction
INFO c.l.s.service.OperationService: publish OperationSavedEvent
DEBUG o.s.orm.jpa.JpaTransactionManager        : Cannot register Spring after-completion synchronization with existing transaction - processing Spring after-completion callbacks immediately, with outcome status 'unknown'
DEBUG o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(1115296438<open>)] after transaction
```

注意，出现了一行日志提示：“Cannot register Spring after-completion synchronization with existing transaction - processing Spring after-completion callbacks immediately, with outcome status 'unknown'”。

顺藤摸瓜进入 JpaTransactionManager 类，其实这一行日志的打印是在它的抽象父类中，即 AbstractPlatformTransactionManager.registerAfterCompletionWithExistingTransaction()

![](https://us1.myximage.com/2021/03/20/50e6675f0228449da017018aa3dd6d30.jpg)

可以看到这里指定了事务状态为 STATUS_UNKNOWN，所以后续的回调逻辑里不再执行事务操作了。这个方法是在 AbstractPlatformTransactionManager.triggerAfterCompletion() 内被调用的：

![](https://us1.myximage.com/2021/03/20/731371a90e79e65ae2ae88741fb79039.jpg)

在这里判断了事务的状态，此时我们的事务状态为有事务，但不是一个新事务，所以进了第二个判断分支。而触发的地方，就是 AbstractPlatformTransactionManager.processCommit()，也就是 Spring 处理事务提交的地方：

```
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    try {

        //... 省略 doCommit 相关逻辑

        try {
            triggerAfterCommit(status);
        }
        finally {
            // ①
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
        }

    }
    finally {
        // ②
        cleanupAfterCompletion(status);
    }
}
```

在 commit 逻辑处理完成后，即标识①的位置，触发了事务提交后的回调。

看到这里，问题已经很清楚了，Spring 在事务提交后，会触发后续回调逻辑，但是如果回调逻辑里也存在事务方法，却又不是一个新事务时，这个妄想加入的事务不会被提交。

## 问题解决

其实明白了问题，解决方案自然也很简单，只需要调整一下事务的传播类型，把保存操作记录的方法，标示为一个新的事务就好了：

```
public class OperationService {

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void saveOperation(String orderNo, String info) {
        Operation operation = new Operation(orderNo, info);
        operationRepository.save(operation);
        log.info("publish OperationSavedEvent");
        applicationContext.publishEvent(new OperationSavedEvent(orderNo, info));
    }

}
```

这样子，操作记录的保存就能写入数据库，而且也能触发后续的事件监听。

## One More Thing

且慢，我们再回想一下，Spring 的事件监听机制，其实是基于观察者模式的同步回调，而事务事件的监听同理，也是在事务提交后，获取事务同步注册器中已经注册了的回调，再同步执行。

刚才分析了 AbstractPlatformTransactionManager.processCommit()，触发回调方法 triggerAfterCompletion() 之后，还有最后一步操作 cleanupAfterCompletion()，即标识②所在的位置。

而在这一步中，才会关闭数据库的连接。

你是不是意识到了什么？

如果在事务事件监听的同步处理中，是个耗时较长的操作，就会一直持有这个数据库连接，线上如果有大量的并发调用，数据库的连接池很容易被耗尽。

想要解决这个问题，可以考虑异步，用新线程去处理这个耗时调用，提前结束回调并释放之前的数据库连接。

## 总结

在这篇文章中，我们分析了在使用 Spring 的事务监听器时，因为原事务已提交，后续事务加入失败而导致的事务失效问题，解决方案就是将后续事务作为新事物处理。

同时梳理了一下 Spring 事务提交和后续处理的过程，明白了回调操作仍然持有之前的数据库连接，如果耗时过长可能会耗尽连接池，可以通过新线程处理来避免这个问题。