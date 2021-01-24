---
title: "Mockito饮用指南"
date: 2021-01-24T16:07:02+08:00
draft: false
---

# 介绍

Mockito是一个用于Java的开源测试mock框架，提供了非常清爽、简洁的API，这个名字来源于经典鸡尾酒 Mojito。

什么是mock？mock就是模拟，有了一个类或接口的定义，我们可以创建一个模拟对象来模拟它的行为，从而就不需要提供这个类或接口的真实实现。
这样在写单元测试的时候，我们只需要mock其他依赖，假设它们的预期返回，就可以专注于测试自己的实现逻辑。

听起来还不错吧，赶紧来尝一口试试。

# 先尝一口

## 引入 Mockito

Mockito支持使用Gradle、Maven、Jar包引入，如果使用Spring Boot的话，spring-boot-starter-test默认已经集成了Mockito。

下文使用的Mockito，版本为3.6.0，项目代码基于Spring Boot。

## 入门操作

先来看一下官方文档上最简单的两个栗子

验证交互：

```
import static org.mockito.Mockito.*;

// mock创建一个 List
List mockedList = mock(List.class);

// 调用mock对象的方法
mockedList.add("one");
mockedList.clear();

// 可以直接验证方法被调用了
verify(mockedList).add("one");
verify(mockedList).clear();
```

mock调用返回：

```
// mock创建一个 LinkedList
LinkedList mockedList = mock(LinkedList.class);

// 使用stub，假设mockedList.get(0)被调用时，会返回"first"
when(mockedList.get(0)).thenReturn("first");

// 控制台会打印"first"
System.out.println(mockedList.get(0));

// 控制台会打印"null"，因为我们没有假设get(999)的返回值
System.out.println(mockedList.get(999));

```

怎么样，看起来是不是很简单，语法也很贴近自然语言。

Mockito本质上是代理模式的应用，mock就是创建proxy对象，在proxy被调用前，使用stub的方式设置返回值，proxy还能记录并跟踪行为。


# 饮用搭配

## doSomething()

void方法或者spy对象，在mock行为时需要使用doThrow()、doAnswer()、doNothing()、doReturn()、doCallRealMethod()

注意区分mock和spy，mock就是完全代理，spy则是部分mock，可以调用真实方法，同时也能被跟踪验证

```
List list = new LinkedList();
List spy = spy(list);

// 这里会抛出IndexOutOfBoundsException，因为调用了真实方法，而list实际上是个空列表
when(spy.get(0)).thenReturn("foo");

// 使用doReturn()来设置spy.get(0)的模拟返回
doReturn("foo").when(spy).get(0);
```

## 使用ArgumentCapture来捕获并验证入参

仅验证行为还不够，如果还需要验证接口调用入参，可以使用ArgumentCapture

```
// 初始化ArgumentCaptor
ArgumentCaptor<Order> orderArgumentCaptor = ArgumentCaptor.forClass(Order.class);

// 在调用处传入ArgumentCaptor.capture()
verify(orderDAO, timeout(1)).update(orderArgumentCaptor.capture());

// orderArgumentCaptor捕获了刚才调用时传入的order参数，通过getValue()获取具体参数对象
Order updatedOrder = orderArgumentCaptor.getValue();

assertEquals(updatedOrder.getStatus(), "processing");
```

## 实用注解

使用@Mock、@Spy、@InjectMocks注解，可以很方便地mock对象并注入

@InjectMocks会尝试通过构造器、setter、字段注入使用@Mock、@Spy注解标识的对象

```
public class MockitoTest {

    /**
     * PostService依赖了PostRepository
     * 使用注解mock了PostRepository，并注入PostService中，不再需要手动设置
     */
    @Mock
    PostRepository postRepository;
    @InjectMocks
    PostService postService;

}
```

@Captor注解可以方便地创建ArgumentCaptor

```
@Captor
ArgumentCaptor<Order> orderArgumentCaptor;

@Test
public void test() {

    /**
     * @Captor可以替代以下写法:
     * ArgumentCaptor<Order> orderArgumentCaptor = ArgumentCaptor.forClass(Order.class);
     */
    verify(orderDAO, timeout(1)).update(orderArgumentCaptor.capture());

}
```

## BDD

BDD即行为驱动开发（Behavior-Driven Development），按照格式描述用例场景，Mockito提供了BDD的语法糖，可以很直接地按BDD语法(given when then)写test case

```
import static org.mockito.BDDMockito.*;

Seller seller = mock(Seller.class);
Shop shop = new Shop(seller);

public void shouldBuyBread() throws Exception {
    //given
    given(seller.askForBread()).willReturn(new Bread());

    //when
    Goods goods = shop.buyBread();

    //then
    assertThat(goods, containBread());
}
```


# 实践一下

一个很简单的业务场景，OrderService提供下订单功能，通过OrderDAO查询并更新Order状态，然后调用OperateLogService保存操作信息。

```

/**
 * 订单类
 */
@Data
public class Order {
    private Long id;
    private String status;
}

/**
 * 订单DAO接口，定义了查询、更新方法
 */
public interface OrderDAO {
    Order findById(Long orderId);
    void update(Order order);
}

/**
 * 操作记录服务，定义了增加操作记录方法
 */
public interface OperateLogService {
    void addRecord(Long orderId, String message);
}

/**
 * 订单服务
 */
public class OrderService {

    private OperateLogService operateLogService;
    private OrderDAO orderDAO;

    @Autowired
    public void setOperateLogService(OperateLogService operateLogService) {
        this.operateLogService = operateLogService;
    }

    @Autowired
    public void setOrderDAO(OrderDAO orderDAO) {
        this.orderDAO = orderDAO;
    }

    /**
     * 下订单操作，更新订单状态并保存，添加操作记录
     * @param orderId 订单id
     */
    public void placeOrder(Long orderId) {
        Order order = orderDAO.findById(orderId);
        if (order == null) {
            throw new RuntimeException("订单不存在");
        }
        order.setStatus("processing");
        orderDAO.update(order);
        operateLogService.addRecord(orderId, "创建订单");
    }
}

```

这个例子中，我们需要为OrderService.placeOrder()这个方法写单元测试。

分析一下实现逻辑，查询order后需要判断order是否存在，这里就需要一个单独的test case来验证异常抛出；

orderDAO.update()是需要验证的重点，该方法需要被正常调用，同时订单的状态会变更为"processing"；OperateLogService是第三方服务，在下单的逻辑中不需要关心其具体的实现逻辑，直接mock，只需要验证被正常调用即可。

```
import static org.junit.Assert.*;
import static org.mockito.Mockito.*;
import static org.mockito.BDDMockito.*;

public class OrderServiceTest {

    @Mock
    OrderDAO orderDAO;
    @Mock
    OperateLogService operateLogService;
    @InjectMocks
    OrderService orderService;

    @Captor
    ArgumentCaptor<Order> orderArgumentCaptor;

    @Test
    public void place_order_should_throw_exception_when_order_not_exist() {
        // given
        // mock查询方法返回空值
        given(orderDAO.findById(anyLong()))
                .willReturn(null);

        // when
        // then
        // 验证会抛出异常
        RuntimeException exception = assertThrows(RuntimeException.class, () -> orderService.placeOrder(1L));
        assertEquals(exception.getMessage(), "订单不存在");

    }

    @Test
    public void place_order_should_update_order_status_then_save() {
        // given
        // mock查询方法返回一个order
        long orderId = 1L;
        Order order = new Order();
        order.setId(orderId);
        order.setStatus("init");
        given(orderDAO.findById(orderId))
                .willReturn(order);

        // when
        // 下订单操作
        orderService.placeOrder(1L);

        // then
        // 验证orderDAO.update被调用一次
        then(orderDAO).should(times(1)).update(orderArgumentCaptor.capture());
        // 验证订单状态被更新
        Order updatedOrder = orderArgumentCaptor.getValue();
        assertEquals(updatedOrder.getId(), orderId);
        assertEquals(updatedOrder.getStatus(), "processing");
        // 验证保存了操作信息
        then(operateLogService).should().addRecord(eq(orderId), anyString());
    }
}

```

这个单元测试中使用了之前提到的BDD、注解和参数捕获器，测试代码清晰易懂，写起来也很方便。

# 再加点料

## 搭配AssertJ风味更佳

相比于JUnit的断言，AssertJ提供了更流畅的断言API，而且方便扩展

```

import static org.assertj.core.api.Assertions.*;

assertEquals(expected, actual);
->
assertThat(actual).isEqualTo(expected);
assertThatThrownBy(() -> {methodWillThrow();})
    .isInstanceOf(SomeException.class)
    .hasMessage("参数不能为空");
```

## PowerMockito

Mockito不支持mock测试static方法、构造器、final方法、private方法等，可以使用PowerMock来解决。

PowerMock也提供了PowerMockito来配合Mockito的API使用。

注: Mockito 2.1.0已支持mock final methods/class

```
orderService = PowerMockito.spy(orderService);
PowerMockito.doReturn(any()).when(orderService, "privateMethod", any());
PowerMockito.verifyPrivate(orderService).invoke("privateMethod", any());
```

## TDD

TDD即测试驱动开发（Test-Driven Development）。

回顾一下常见的开发流程，先需求分析，然后分解任务，再实现具体逻辑，最后写单测验证。把单元测试放在最后一步，容易遗漏琐碎的改动点，而且一些设计上的问题暴露地比较晚，随着需求不断迭代，开发人员也很容易产生惰性，“忘记”写单元测试。

回过来看整个流程，在分解任务的时候，其实就是实现具体逻辑前的的设计过程；这个阶段会设计需要的类、方法以及他们之间的交互，这些设计方案应该是落地的，并且是可验证的。

因此可以采用TDD，把编写测试用例的流程前提至设计阶段，分解后的任务单元，其实就是一个或多个可验证的测试用例，在编写测试用例的时候，可以”强迫“自己思考设计的合理性，然后立刻实现并验证。

TDD时，保持测试->实现->重构(红->绿->黄)的节奏，建议在IDE上配好顺手的rerun test的快捷键。

# 总结

Mockito简明易懂的API，结合上文提到的一些实践，写单测不再费时费力，赶紧试试吧。

🍻 cheers~

 ---

 # 参考

 [Mockito官方文档](https://site.mockito.org/)

 [AssertJ官方文档](http://joel-costigliola.github.io/assertj/index.html)

 [PowerMock官方文档](https://github.com/powermock/powermock/wiki)
