title: TCC-Transaction 源码分析 —— 项目实战
date: 2018-03-15
tags:
categories: TCC-Transaction
permalink: TCC-Transaction/http-sample

---

摘要: 原创出处 http://www.iocoder.cn/TCC-Transaction/http-sample/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 TCC-Transaction 1.2.3.3 正式版**  

- [1. 概述](http://www.iocoder.cn/TCC-Transaction/http-sample/)
- [2. 实体结构](http://www.iocoder.cn/TCC-Transaction/http-sample/)
  - [2.1 商城服务](http://www.iocoder.cn/TCC-Transaction/http-sample/)
  - [2.2 资金服务](http://www.iocoder.cn/TCC-Transaction/http-sample/)
  - [2.3 红包服务](http://www.iocoder.cn/TCC-Transaction/http-sample/)
- [3. 服务调用](http://www.iocoder.cn/TCC-Transaction/http-sample/)
- [4. 下单支付流程](http://www.iocoder.cn/TCC-Transaction/http-sample/)
  - [4.1 Try 阶段](http://www.iocoder.cn/TCC-Transaction/http-sample/)
  - [4.2 Confirm / Cancel 阶段](http://www.iocoder.cn/TCC-Transaction/http-sample/)
    - [4.2.1 Confirm](http://www.iocoder.cn/TCC-Transaction/http-sample/)
    - [4.2.2 Cancel](http://www.iocoder.cn/TCC-Transaction/http-sample/)
- [666. 彩蛋](http://www.iocoder.cn/TCC-Transaction/http-sample/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

---

# 1. 概述

本文分享 **TCC 项目实战**。以官方 Maven项目 `tcc-transaction-http-sample` 为例子( `tcc-transaction-dubbo-sample` 类似 )。

建议你已经成功启动了该项目。如果不知道如何启动，可以先查看[《TCC-Transaction 源码分析 —— 调试环境搭建》](http://www.iocoder.cn/TCC-Transaction/build-debugging-environment/?self)。如果再碰到问题，欢迎加微信公众号( **芋道源码** )，我会一一仔细回复。

OK，首先我们简单了解下这个项目。

![](http://www.iocoder.cn/images/TCC-Transaction/2018_03_15/01.png)

* 首页 => 商品列表 => 确认支付页 => 支付结果页
* 使用账户余额 + 红包余额**联合**支付购买商品，并账户之间**转账**。

项目拆分三个子 Maven 项目：

* `tcc-transaction-http-order` ：商城服务，提供商品和商品订单逻辑。
* `tcc-transaction-http-capital` ：资金服务，提供账户余额逻辑。
* `tcc-transaction-http-redpacket` ：红包服务，提供红包余额逻辑。

![](http://www.iocoder.cn/images/TCC-Transaction/2018_03_15/03.png)

> 你行好事会因为得到赞赏而愉悦  
> 同理，开源项目贡献者会因为 Star 而更加有动力  
> 为 TCC-Transaction 点赞！[传送门](https://github.com/changmingxie/tcc-transaction)

# 2. 实体结构

## 2.1 商城服务

![](http://www.iocoder.cn/images/TCC-Transaction/2018_03_15/02.png)

* Shop，商店表。实体代码如下：

    ```Java
    public class Shop {
    
        /**
         * 商店编号
         */
        private long id;
        /**
         * 所有者用户编号
         */
        private long ownerUserId;
    }
    ```

* Product，商品表。实体代码如下：

    ```Java
    public class Product implements Serializable {
    
        /**
         * 商品编号
         */
        private long productId;
        /**
         * 商店编号
         */
        private long shopId;
        /**
         * 商品名
         */
        private String productName;
        /**
         * 单价
         */
        private BigDecimal price;
    }
    ```

* Order，订单表。实现代码如下：

    ```Java
    public class Order implements Serializable {
    
        private static final long serialVersionUID = -5908730245224893590L;
    
        /**
         * 订单编号
         */
        private long id;
        /**
         * 支付( 下单 )用户编号
         */
        private long payerUserId;
        /**
         * 收款( 商店拥有者 )用户编号
         */
        private long payeeUserId;
        /**
         * 红包支付金额
         */
        private BigDecimal redPacketPayAmount;
        /**
         * 账户余额支付金额
         */
        private BigDecimal capitalPayAmount;
        /**
         * 订单状态
         * - DRAFT ：草稿
         * - PAYING ：支付中
         * - CONFIRMED ：支付成功
         * - PAY_FAILED ：支付失败
         */
        private String status = "DRAFT";
        /**
         * 商户订单号，使用 UUID 生成
         */
        private String merchantOrderNo;
    
        /**
         * 订单明细数组
         * 非存储字段
         */
        private List<OrderLine> orderLines = new ArrayList<OrderLine>();
    }
    ```
    
* OrderLine，订单明细。实体代码如下：

    ```Java
    public class OrderLine implements Serializable {
    
        private static final long serialVersionUID = 2300754647209250837L;
    
        /**
         * 订单编号
         */
        private long id;
        /**
         * 商品编号
         */
        private long productId;
        /**
         * 数量
         */
        private int quantity;
        /**
         * 单价
         */
        private BigDecimal unitPrice;
    }
    ```

**业务逻辑**：

下单时，插入订单状态为 `"DRAFT"` 的订单( Order )记录，并插入购买的商品订单明细( OrderLine )记录。支付时，更新订单状态为 `"PAYING"`。

* 订单支付成功，更新订单状态为 `"CONFIRMED"`。
* 订单支付失败，更新订单状体为 `"PAY_FAILED"`。

## 2.2 资金服务

关系较为简单，有两个实体：

* CapitalAccount，资金账户余额。实体代码如下：
 
    ```Java
    public class CapitalAccount {
    
        /**
         * 账户编号
         */
        private long id;
        /**
         * 用户编号
         */
        private long userId;
        /**
         * 余额
         */
        private BigDecimal balanceAmount;
    }
    ```

* TradeOrder，交易订单表。实体代码如下：

    ```Java
    public class TradeOrder {
    
        /**
         * 交易订单编号
         */
        private long id;
        /**
         * 转出用户编号
         */
        private long selfUserId;
        /**
         * 转入用户编号
         */
        private long oppositeUserId;
        /**
         * 商户订单号
         */
        private String merchantOrderNo;
        /**
         * 金额
         */
        private BigDecimal amount;
        /**
         * 交易订单状态
         * - DRAFT ：草稿
         * - CONFIRM ：交易成功
         * - CANCEL ：交易取消
         */
        private String status = "DRAFT";
    }
    ```

**业务逻辑**：

订单支付支付中，插入交易订单状态为 `"DRAFT"` 的订单( TradeOrder )记录，并更新**减少**下单用户的资金账户余额。

* 订单支付成功，更新交易订单状态为 `"CONFIRM"`，并更新**增加**商店拥有用户的资金账户余额。
* 订单支付失败，更新交易订单状态为 `"CANCEL"`，并更新**增加( 恢复 )**下单用户的资金账户余额。

## 2.3 红包服务

关系较为简单，**和资金服务 99.99% 相同**，有两个实体：

* RedPacketAccount，红包账户余额。实体代码如下：
 
    ```Java
    public class RedPacketAccount {
    
        /**
         * 账户编号
         */
        private long id;
        /**
         * 用户编号
         */
        private long userId;
        /**
         * 余额
         */
        private BigDecimal balanceAmount;
    }
    ```

* TradeOrder，交易订单表。实体代码如下：

    ```Java
    public class TradeOrder {
    
        /**
         * 交易订单编号
         */
        private long id;
        /**
         * 转出用户编号
         */
        private long selfUserId;
        /**
         * 转入用户编号
         */
        private long oppositeUserId;
        /**
         * 商户订单号
         */
        private String merchantOrderNo;
        /**
         * 金额
         */
        private BigDecimal amount;
        /**
         * 交易订单状态
         * - DRAFT ：草稿
         * - CONFIRM ：交易成功
         * - CANCEL ：交易取消
         */
        private String status = "DRAFT";
    }
    ```

**业务逻辑**：

订单支付支付中，插入交易订单状态为 `"DRAFT"` 的订单( TradeOrder )记录，并更新**减少**下单用户的红包账户余额。

* 订单支付成功，更新交易订单状态为 `"CONFIRM"`，并更新**增加**商店拥有用户的红包账户余额。
* 订单支付失败，更新交易订单状态为 `"CANCEL"`，并更新**增加( 恢复 )**下单用户的红包账户余额。

# 3. 服务调用

服务之间，通过 **HTTP** 进行调用。

**红包服务和资金服务为商城服务提供调用( 以资金服务为例子 )**：

* XML 配置如下 ：

    ```XML
    // appcontext-service-provider.xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:util="http://www.springframework.org/schema/util"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd">
    
        <bean name="capitalAccountRepository"
              class="org.mengyun.tcctransaction.sample.http.capital.domain.repository.CapitalAccountRepository"/>
    
        <bean name="tradeOrderRepository"
              class="org.mengyun.tcctransaction.sample.http.capital.domain.repository.TradeOrderRepository"/>
    
        <bean name="capitalTradeOrderService"
              class="org.mengyun.tcctransaction.sample.http.capital.service.CapitalTradeOrderServiceImpl"/>
    
        <bean name="capitalAccountService"
              class="org.mengyun.tcctransaction.sample.http.capital.service.CapitalAccountServiceImpl"/>
    
        <bean name="capitalTradeOrderServiceExporter"
              class="org.springframework.remoting.httpinvoker.SimpleHttpInvokerServiceExporter">
            <property name="service" ref="capitalTradeOrderService"/>
            <property name="serviceInterface"
                      value="org.mengyun.tcctransaction.sample.http.capital.api.CapitalTradeOrderService"/>
        </bean>
    
        <bean name="capitalAccountServiceExporter"
              class="org.springframework.remoting.httpinvoker.SimpleHttpInvokerServiceExporter">
            <property name="service" ref="capitalAccountService"/>
            <property name="serviceInterface"
                      value="org.mengyun.tcctransaction.sample.http.capital.api.CapitalAccountService"/>
        </bean>
    
    
        <bean id="httpServer"
              class="org.springframework.remoting.support.SimpleHttpServerFactoryBean">
            <property name="contexts">
                <util:map>
                    <entry key="/remoting/CapitalTradeOrderService" value-ref="capitalTradeOrderServiceExporter"/>
                    <entry key="/remoting/CapitalAccountService" value-ref="capitalAccountServiceExporter"/>
                </util:map>
            </property>
            <property name="port" value="8081"/>
        </bean>
    
    </beans>
    ```

* Java 代码实现如下 ：

    ```Java
    public class CapitalAccountServiceImpl implements CapitalAccountService {
        
        @Autowired
        CapitalAccountRepository capitalAccountRepository;
    
        @Override
        public BigDecimal getCapitalAccountByUserId(long userId) {
            return capitalAccountRepository.findByUserId(userId).getBalanceAmount();
        }
    
    }
    
    public class CapitalAccountServiceImpl implements CapitalAccountService {
    
        @Autowired
        CapitalAccountRepository capitalAccountRepository;
    
        @Override
        public BigDecimal getCapitalAccountByUserId(long userId) {
            return capitalAccountRepository.findByUserId(userId).getBalanceAmount();
        }
    
    }
    ```

-------

**商城服务调用**

* XML 配置如下：

    ```XML
    // appcontext-service-consumer.xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    
        <bean id="httpInvokerRequestExecutor"
              class="org.springframework.remoting.httpinvoker.CommonsHttpInvokerRequestExecutor">
            <property name="httpClient">
                <bean class="org.apache.commons.httpclient.HttpClient">
                    <property name="httpConnectionManager">
                        <ref bean="multiThreadHttpConnectionManager"/>
                    </property>
                </bean>
            </property>
        </bean>
    
        <bean id="multiThreadHttpConnectionManager"
              class="org.apache.commons.httpclient.MultiThreadedHttpConnectionManager">
            <property name="params">
                <bean class="org.apache.commons.httpclient.params.HttpConnectionManagerParams">
                    <property name="connectionTimeout" value="200000"/>
                    <property name="maxTotalConnections" value="600"/>
                    <property name="defaultMaxConnectionsPerHost" value="512"/>
                    <property name="soTimeout" value="5000"/>
                </bean>
            </property>
        </bean>
    
        <bean id="captialTradeOrderService" class="org.springframework.remoting.httpinvoker.HttpInvokerProxyFactoryBean">
            <property name="serviceUrl" value="http://localhost:8081/remoting/CapitalTradeOrderService"/>
            <property name="serviceInterface"
                      value="org.mengyun.tcctransaction.sample.http.capital.api.CapitalTradeOrderService"/>
            <property name="httpInvokerRequestExecutor" ref="httpInvokerRequestExecutor"/>
        </bean>
    
        <bean id="capitalAccountService" class="org.springframework.remoting.httpinvoker.HttpInvokerProxyFactoryBean">
            <property name="serviceUrl" value="http://localhost:8081/remoting/CapitalAccountService"/>
            <property name="serviceInterface"
                      value="org.mengyun.tcctransaction.sample.http.capital.api.CapitalAccountService"/>
            <property name="httpInvokerRequestExecutor" ref="httpInvokerRequestExecutor"/>
        </bean>
    
        <bean id="redPacketAccountService" class="org.springframework.remoting.httpinvoker.HttpInvokerProxyFactoryBean">
            <property name="serviceUrl" value="http://localhost:8082/remoting/RedPacketAccountService"/>
            <property name="serviceInterface"
                      value="org.mengyun.tcctransaction.sample.http.redpacket.api.RedPacketAccountService"/>
            <property name="httpInvokerRequestExecutor" ref="httpInvokerRequestExecutor"/>
        </bean>
    
        <bean id="redPacketTradeOrderService" class="org.springframework.remoting.httpinvoker.HttpInvokerProxyFactoryBean">
            <property name="serviceUrl" value="http://localhost:8082/remoting/RedPacketTradeOrderService"/>
            <property name="serviceInterface"
                      value="org.mengyun.tcctransaction.sample.http.redpacket.api.RedPacketTradeOrderService"/>
            <property name="httpInvokerRequestExecutor" ref="httpInvokerRequestExecutor"/>
        </bean>
    
    </beans>
    ```

* Java 接口接口如下：

    ```Java
    public interface CapitalAccountService {
        BigDecimal getCapitalAccountByUserId(long userId);
    }
    
    public interface CapitalTradeOrderService {
        String record(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto);
    }
    
    public interface RedPacketAccountService {
        BigDecimal getRedPacketAccountByUserId(long userId);
    }
    
    public interface RedPacketTradeOrderService {
        String record(TransactionContext transactionContext, RedPacketTradeOrderDto tradeOrderDto);
    }
    ```

# 4. 下单支付流程

**ps**：数据访问的方法，请自己拉取代码，使用 IDE 查看。谢谢。🙂

下单支付流程，整体流程如下图( [打开大图](./../../images/TCC-Transaction/2018_03_15/04.png) )：

![](http://www.iocoder.cn/images/TCC-Transaction/2018_03_15/04.png)

点击**【支付】**按钮，下单支付流程。实现代码如下：

```Java
@Controller
@RequestMapping("")
public class OrderController {
    
        @RequestMapping(value = "/placeorder", method = RequestMethod.POST)
    public ModelAndView placeOrder(@RequestParam String redPacketPayAmount,
                                   @RequestParam long shopId,
                                   @RequestParam long payerUserId,
                                   @RequestParam long productId) {
        PlaceOrderRequest request = buildRequest(redPacketPayAmount, shopId, payerUserId, productId);
        // 下单并支付订单
        String merchantOrderNo = placeOrderService.placeOrder(request.getPayerUserId(), request.getShopId(),
                request.getProductQuantities(), request.getRedPacketPayAmount());
        // 返回
        ModelAndView mv = new ModelAndView("pay_success");
        // 查询订单状态
        String status = orderService.getOrderStatusByMerchantOrderNo(merchantOrderNo);
        // 支付结果提示
        String payResultTip = null;
        if ("CONFIRMED".equals(status)) {
            payResultTip = "支付成功";
        } else if ("PAY_FAILED".equals(status)) {
            payResultTip = "支付失败";
        }
        mv.addObject("payResult", payResultTip);
        // 商品信息
        mv.addObject("product", productRepository.findById(productId));
        // 资金账户金额 和 红包账户金额
        mv.addObject("capitalAmount", accountService.getCapitalAccountByUserId(payerUserId));
        mv.addObject("redPacketAmount", accountService.getRedPacketAccountByUserId(payerUserId));
        return mv;
    }

}
```

* 调用 `PlaceOrderService#placeOrder(...)` 方法，下单并支付订单。
* 调用 [`OrderService#getOrderStatusByMerchantOrderNo(...)`](https://github.com/YunaiV/tcc-transaction/blob/ff443d9e6d39bd798ed042034bad123b83675922/tcc-transaction-tutorial-sample/tcc-transaction-http-sample/tcc-transaction-http-order/src/main/java/org/mengyun/tcctransaction/sample/http/order/domain/service/OrderServiceImpl.java) 方法，查询订单状态。

-------

调用 `PlaceOrderService#placeOrder(...)` 方法，下单并支付订单。实现代码如下：

```Java
@Service
public class PlaceOrderServiceImpl {

    public String placeOrder(long payerUserId, long shopId, List<Pair<Long, Integer>> productQuantities, BigDecimal redPacketPayAmount) {
        // 获取商店
        Shop shop = shopRepository.findById(shopId);
        // 创建订单
        Order order = orderService.createOrder(payerUserId, shop.getOwnerUserId(), productQuantities);
        // 发起支付
        Boolean result = false;
        try {
            paymentService.makePayment(order, redPacketPayAmount, order.getTotalAmount().subtract(redPacketPayAmount));
        } catch (ConfirmingException confirmingException) {
            // exception throws with the tcc transaction status is CONFIRMING,
            // when tcc transaction is confirming status,
            // the tcc transaction recovery will try to confirm the whole transaction to ensure eventually consistent.
            result = true;
        } catch (CancellingException cancellingException) {
            // exception throws with the tcc transaction status is CANCELLING,
            // when tcc transaction is under CANCELLING status,
            // the tcc transaction recovery will try to cancel the whole transaction to ensure eventually consistent.
        } catch (Throwable e) {
            // other exceptions throws at TRYING stage.
            // you can retry or cancel the operation.
            e.printStackTrace();
        }
        return order.getMerchantOrderNo();
    }

}
```

* 调用 `ShopRepository#findById(...)` 方法，查询商店。
* 调用 `OrderService#createOrder(...)` 方法，创建订单状态为 `"DRAFT"` 的**商城**订单。实际业务不会这么做，此处仅仅是例子，简化流程。实现代码如下：

    ```Java
    @Service
    public class OrderServiceImpl {
    
        @Transactional
        public Order createOrder(long payerUserId, long payeeUserId, List<Pair<Long, Integer>> productQuantities) {
            Order order = orderFactory.buildOrder(payerUserId, payeeUserId, productQuantities);
            orderRepository.createOrder(order);
            return order;
        }
    
    }
    ```
    
* 调用 `PaymentService#makePayment(...)` 方法，发起支付，**TCC 流程**。
* **生产代码对于异常需要进一步处理**。
* **生产代码对于异常需要进一步处理**。
* **生产代码对于异常需要进一步处理**。

## 4.1 Try 阶段

**商城服务**

调用 `PaymentService#makePayment(...)` 方法，发起 Try 流程，实现代码如下：

```Java
@Compensable(confirmMethod = "confirmMakePayment", cancelMethod = "cancelMakePayment")
@Transactional
public void makePayment(Order order, BigDecimal redPacketPayAmount, BigDecimal capitalPayAmount) {
   System.out.println("order try make payment called.time seq:" + DateFormatUtils.format(Calendar.getInstance(), "yyyy-MM-dd HH:mm:ss"));
   // 更新订单状态为支付中
   order.pay(redPacketPayAmount, capitalPayAmount);
   orderRepository.updateOrder(order);
   // 资金账户余额支付订单
   String result = tradeOrderServiceProxy.record(null, buildCapitalTradeOrderDto(order));
   // 红包账户余额支付订单
   String result2 = tradeOrderServiceProxy.record(null, buildRedPacketTradeOrderDto(order));
}
```

* 设置方法注解 @Compensable
    * 事务传播级别 Propagation.REQUIRED ( **默认值** )
    * 设置 `confirmMethod` /  `cancelMethod` 方法名
    * 事务上下文编辑类 DefaultTransactionContextEditor ( **默认值** )

* 设置方法注解 @Transactional，保证方法操作原子性。

* 调用 `OrderRepository#updateOrder(...)` 方法，更新订单状态为**支付中**。实现代码如下：

    ```Java
    // Order.java
    public void pay(BigDecimal redPacketPayAmount, BigDecimal capitalPayAmount) {
       this.redPacketPayAmount = redPacketPayAmount;
       this.capitalPayAmount = capitalPayAmount;
       this.status = "PAYING";
    }
    ```
    
* 调用 `TradeOrderServiceProxy#record(...)` 方法，**资金**账户余额支付订单。实现代码如下：

    ```Java
    // TradeOrderServiceProxy.java
    @Compensable(propagation = Propagation.SUPPORTS, confirmMethod = "record", cancelMethod = "record", transactionContextEditor = Compensable.DefaultTransactionContextEditor.class)
    public String record(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto) {
       return capitalTradeOrderService.record(transactionContext, tradeOrderDto);
    }
    
    // CapitalTradeOrderService.java
    public interface CapitalTradeOrderService {
        String record(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto);
    }
    ```
    * 设置方法注解 @Compensable
        * `propagation=Propagation.SUPPORTS` ：支持当前事务，如果当前没有事务，就以非事务方式执行。**为什么不使用 REQUIRED** ？如果使用 REQUIRED 事务传播级别，事务恢复重试时，会发起新的事务。
        * `confirmMethod`、`cancelMethod` 使用和 try 方法**相同方法名**：**本地发起**远程服务 TCC confirm / cancel 阶段，调用相同方法进行事务的提交或回滚。远程服务的 CompensableTransactionInterceptor 会根据事务的状态是 CONFIRMING / CANCELLING 来调用对应方法。
   
    * 调用 `CapitalTradeOrderService#record(...)` 方法，远程调用，发起**资金**账户余额支付订单。
        * 本地方法调用时，参数 `transactionContext` 传递 `null` 即可，TransactionContextEditor 会设置。在[《TCC-Transaction 源码分析 —— TCC 实现》「6.3 资源协调者拦截器」](http://www.iocoder.cn/TCC-Transaction/tcc-core/?self)有详细解析。
        * 远程方法调用时，参数 `transactionContext` 需要传递。Dubbo 远程方法调用实际也进行了传递，传递方式较为特殊，通过隐式船舱，在[《TCC-Transaction 源码分析 —— Dubbo 支持》「3. Dubbo 事务上下文编辑器」](http://www.iocoder.cn/TCC-Transaction/dubbo-support/?self)有详细解析。

* 调用 `TradeOrderServiceProxy#record(...)` 方法，**红包**账户余额支付订单。和**资金**账户余额支付订单 99.99% 类似，不重复“复制粘贴”。

-------

**资金服务**

调用 `CapitalTradeOrderServiceImpl#record(...)` 方法，**红包**账户余额支付订单。实现代码如下：

```Java
@Override
@Compensable(confirmMethod = "confirmRecord", cancelMethod = "cancelRecord", transactionContextEditor = Compensable.DefaultTransactionContextEditor.class)
@Transactional
public String record(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto) {
   // 调试用
   try {
       Thread.sleep(1000l);
//            Thread.sleep(10000000L);
   } catch (InterruptedException e) {
       throw new RuntimeException(e);
   }
   System.out.println("capital try record called. time seq:" + DateFormatUtils.format(Calendar.getInstance(), "yyyy-MM-dd HH:mm:ss"));
   // 生成交易订单
   TradeOrder tradeOrder = new TradeOrder(
           tradeOrderDto.getSelfUserId(),
           tradeOrderDto.getOppositeUserId(),
           tradeOrderDto.getMerchantOrderNo(),
           tradeOrderDto.getAmount()
   );
   tradeOrderRepository.insert(tradeOrder);
   // 更新减少下单用户的资金账户余额
   CapitalAccount transferFromAccount = capitalAccountRepository.findByUserId(tradeOrderDto.getSelfUserId());
   transferFromAccount.transferFrom(tradeOrderDto.getAmount());
   capitalAccountRepository.save(transferFromAccount);
   return "success";
}
```

* 设置方法注解 @Compensable
    * 事务传播级别 Propagation.REQUIRED ( **默认值** )
    * 设置 `confirmMethod` /  `cancelMethod` 方法名
    * 事务上下文编辑类 DefaultTransactionContextEditor ( **默认值** )

* 设置方法注解 @Transactional，保证方法操作原子性。
* 调用 [`TradeOrderRepository#insert(...)`](https://github.com/YunaiV/tcc-transaction/blob/ff443d9e6d39bd798ed042034bad123b83675922/tcc-transaction-tutorial-sample/tcc-transaction-http-sample/tcc-transaction-http-capital/src/main/java/org/mengyun/tcctransaction/sample/http/capital/domain/repository/TradeOrderRepository.java) 方法，生成订单状态为 `"DRAFT"` 的交易订单。
* 调用 [`CapitalAccountRepository#save(...)`](https://github.com/YunaiV/tcc-transaction/blob/ff443d9e6d39bd798ed042034bad123b83675922/tcc-transaction-tutorial-sample/tcc-transaction-http-sample/tcc-transaction-http-capital/src/main/java/org/mengyun/tcctransaction/sample/http/capital/domain/repository/CapitalAccountRepository.java) 方法，更新减少下单用户的资金账户余额。**Try 阶段锁定资源时，一定要先扣。TCC 是最终事务一致性，如果先添加，可能被使用**。

## 4.2 Confirm / Cancel 阶段

当 Try 操作**全部**成功时，发起 Confirm 操作。  
当 Try 操作存在**任务**失败时，发起 Cancel 操作。

### 4.2.1 Confirm

**商城服务**

调用 `PaymentServiceImpl#confirmMakePayment(...)` 方法，更新订单状态为支付**成功**。实现代码如下：

```Java
public void confirmMakePayment(Order order, BigDecimal redPacketPayAmount, BigDecimal capitalPayAmount) {
   // 调试用
   try {
       Thread.sleep(1000l);
   } catch (InterruptedException e) {
       throw new RuntimeException(e);
   }
   System.out.println("order confirm make payment called. time seq:" + DateFormatUtils.format(Calendar.getInstance(), "yyyy-MM-dd HH:mm:ss"));
   // 更新订单状态为支付成功
   order.confirm();
   orderRepository.updateOrder(order);
}
```

* **生产代码该方法需要加下 @Transactional 注解，保证原子性**。
* 调用 `OrderRepository#updateOrder(...)` 方法，更新订单状态为支付成功。实现代码如下：

    ```Java
    // Order.java
    public void confirm() {
       this.status = "CONFIRMED";
    }
    ```

-------

**资金服务**

调用 `CapitalTradeOrderServiceImpl#confirmRecord(...)` 方法，更新交易订单状态为交易**成功**。

```Java
@Transactional
public void confirmRecord(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto) {
   // 调试用
   try {
       Thread.sleep(1000l);
   } catch (InterruptedException e) {
       throw new RuntimeException(e);
   }
   System.out.println("capital confirm record called. time seq:" + DateFormatUtils.format(Calendar.getInstance(), "yyyy-MM-dd HH:mm:ss"));
   // 查询交易记录
   TradeOrder tradeOrder = tradeOrderRepository.findByMerchantOrderNo(tradeOrderDto.getMerchantOrderNo());
   // 判断交易记录状态。因为 `#record()` 方法，可能事务回滚，记录不存在 / 状态不对
   if (null != tradeOrder && "DRAFT".equals(tradeOrder.getStatus())) {
       // 更新订单状态为交易成功
       tradeOrder.confirm();
       tradeOrderRepository.update(tradeOrder);
       // 更新增加商店拥有者用户的资金账户余额
       CapitalAccount transferToAccount = capitalAccountRepository.findByUserId(tradeOrderDto.getOppositeUserId());
       transferToAccount.transferTo(tradeOrderDto.getAmount());
       capitalAccountRepository.save(transferToAccount);
   }
}
```

* 设置方法注解 @Transactional，保证方法操作原子性。
* **判断交易记录状态**。因为 `#record()` 方法，可能事务回滚，记录不存在 / 状态不对。
* 调用 [`TradeOrderRepository#update(...)`](https://github.com/YunaiV/tcc-transaction/blob/ff443d9e6d39bd798ed042034bad123b83675922/tcc-transaction-tutorial-sample/tcc-transaction-http-sample/tcc-transaction-http-capital/src/main/java/org/mengyun/tcctransaction/sample/http/capital/domain/repository/TradeOrderRepository.java) 方法，更新交易订单状态为交易**成功**。
* 调用 [`CapitalAccountRepository#save(...)`](https://github.com/YunaiV/tcc-transaction/blob/ff443d9e6d39bd798ed042034bad123b83675922/tcc-transaction-tutorial-sample/tcc-transaction-http-sample/tcc-transaction-http-capital/src/main/java/org/mengyun/tcctransaction/sample/http/capital/domain/repository/CapitalAccountRepository.java) 方法，更新增加商店拥有者用户的资金账户余额。实现代码如下：

    ```Java
    // CapitalAccount.java
    public void transferTo(BigDecimal amount) {
       this.balanceAmount = this.balanceAmount.add(amount);
    }
    ```

-------

**红包服务**

和**资源服务** 99.99% 相同，不重复“复制粘贴”。

### 4.2.2 Cancel

**商城服务**

调用 `PaymentServiceImpl#cancelMakePayment(...)` 方法，更新订单状态为支付**失败**。实现代码如下：

```Java
public void cancelMakePayment(Order order, BigDecimal redPacketPayAmount, BigDecimal capitalPayAmount) {
   // 调试用
   try {
       Thread.sleep(1000l);
   } catch (InterruptedException e) {
       throw new RuntimeException(e);
   }
   System.out.println("order cancel make payment called.time seq:" + DateFormatUtils.format(Calendar.getInstance(), "yyyy-MM-dd HH:mm:ss"));
   // 更新订单状态为支付失败
   order.cancelPayment();
   orderRepository.updateOrder(order);
}
```

* **生产代码该方法需要加下 @Transactional 注解，保证原子性**。
* 调用 `OrderRepository#updateOrder(...)` 方法，更新订单状态为支付失败。实现代码如下：

    ```Java
    // Order.java
    public void cancelPayment() {
        this.status = "PAY_FAILED";
    }
    ```

-------

**资金服务**

调用 `CapitalTradeOrderServiceImpl#cancelRecord(...)` 方法，更新交易订单状态为交易**失败**。

```Java
@Transactional
public void cancelRecord(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto) {
   // 调试用
   try {
       Thread.sleep(1000l);
   } catch (InterruptedException e) {
       throw new RuntimeException(e);
   }
   System.out.println("capital cancel record called. time seq:" + DateFormatUtils.format(Calendar.getInstance(), "yyyy-MM-dd HH:mm:ss"));
   // 查询交易记录
   TradeOrder tradeOrder = tradeOrderRepository.findByMerchantOrderNo(tradeOrderDto.getMerchantOrderNo());
   // 判断交易记录状态。因为 `#record()` 方法，可能事务回滚，记录不存在 / 状态不对
   if (null != tradeOrder && "DRAFT".equals(tradeOrder.getStatus())) {
       // / 更新订单状态为交易失败
       tradeOrder.cancel();
       tradeOrderRepository.update(tradeOrder);
       // 更新增加( 恢复 )下单用户的资金账户余额
       CapitalAccount capitalAccount = capitalAccountRepository.findByUserId(tradeOrderDto.getSelfUserId());
       capitalAccount.cancelTransfer(tradeOrderDto.getAmount());
       capitalAccountRepository.save(capitalAccount);
   }
}
```

* 设置方法注解 @Transactional，保证方法操作原子性。
* **判断交易记录状态**。因为 `#record()` 方法，可能事务回滚，记录不存在 / 状态不对。
* 调用 [`TradeOrderRepository#update(...)`](https://github.com/YunaiV/tcc-transaction/blob/ff443d9e6d39bd798ed042034bad123b83675922/tcc-transaction-tutorial-sample/tcc-transaction-http-sample/tcc-transaction-http-capital/src/main/java/org/mengyun/tcctransaction/sample/http/capital/domain/repository/TradeOrderRepository.java) 方法，更新交易订单状态为交易**失败**。
* 调用 [`CapitalAccountRepository#save(...)`](https://github.com/YunaiV/tcc-transaction/blob/ff443d9e6d39bd798ed042034bad123b83675922/tcc-transaction-tutorial-sample/tcc-transaction-http-sample/tcc-transaction-http-capital/src/main/java/org/mengyun/tcctransaction/sample/http/capital/domain/repository/CapitalAccountRepository.java) 方法，更新增加( 恢复 )下单用户的资金账户余额。实现代码如下：

    ```Java
    // CapitalAccount.java
    public void cancelTransfer(BigDecimal amount) {
        transferTo(amount);
    }
    ```

-------

**红包服务**

和**资源服务** 99.99% 相同，不重复“复制粘贴”。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

嘿嘿，代码只是看起来比较多，实际不多。

蚂蚁金融云提供了银行间转账的 TCC 过程例子，有兴趣的同学可以看看：[《蚂蚁金融云 —— 分布式事务服务（DTS） —— 场景介绍》](https://www.cloud.alipay.com/docs/2/46886)。

本系列 EOF ~撒花

胖友，分享个朋友圈，可好？！


