# 订单架构 · LILISHOP-开发者中心
订单架构[](#订单架构)
-------------

#### 订单创建流程图[](#订单创建流程图)

![](https://docs.pickmall.cn/architecture/images/createOrder-flow.png)

##### 说明[](#说明)

> 1、请求创建订单
> 
> 2、判定购物车中选中的商品的可配送区域
> 
> 3、校验每个商品是否有效，库存是否充足
> 
> 5、进入促销模块，进行促销模块的价格渲染
> 
> 6、先进行满优惠活动价格渲染（这里会将满减或者满折的金额，根据商品价格，分别拆分到每个商品的价格明细中）
> 
> 7、单品优惠活动价格渲染，重新计算单商品的服务费用，满优惠费用，优惠券优惠费用，单品优惠费用等等
> 
> 8、获取购物车中选中的所有优惠券
> 
> 9、将单品优惠券，优惠价格计入单品，满足多商品的优惠券，分别按照商品成交价格的比例进行分配，以及平台优惠券跨店铺的商品优惠价格渲染（这里在设计时，优惠券加入购物车时，购物车缓存中会存储优惠券与商品的多对多关系，这样可以在价格计算时，快速计算优惠券分别需要优惠哪几件商品）
> 
> 10、将商品金额，店铺金额等等进行汇总返回
> 
> 11、运费计算
> 
> 13、返回订单信息，前端跳转收银台页面

库存扣减问题：在此架构中，商品的库存扣减是发生在付款后，如果订单已经付款，发现库存不足，会自动走系统取消订单流程

#### 订单付款流程图[](#订单付款流程图)

![](https://docs.pickmall.cn/architecture/images/payOrder-flow.png)

##### 说明[](#说明)

> 1、用户请求付款到收银台
> 
> 2、根据用户请求的信息，以及客户端，返回支付的表单，或者支付的跳转链接
> 
> 3、前往第三方进行支付
> 
> 4、返回支付结果
> 
> 5、等待第三方通知api支付结果
> 
> 6、支付成功后，自动操作订单状态到已支付，且发送mq消息，告知mq订单信息
> 
> 9、商品库存扣减，促销商品库存扣件（这里用的是redis的脚步执行批量库存扣减，如果成功，则订单进入下一流程，如果库存扣减某一环节失败，则自动发起订单退款操作）
> 
> 10、将订单信息修改为待发货

##### 订单付款问题补充[](#订单付款问题补充)

这里架构时，发现入口只有一个，但是库存扣减的时机却有好多个。

1.  加入购物车库存扣减，这样做的好处是，会员不会到了结算页面，或者付款页面才知道商品数量不足，但是这样会带来额外的性能损耗，以及有人恶意加入购物车问题，无法操作。解决恶意操作购物车，那么就需要定时清理所有用户的购物车，不合逻辑所以排除、
2.  下单后，支付前，库存扣减。这样会存在一个问题，用户在下单后无法第一时间进行付款，如果并发请求后，这块可能等待几秒甚至几分钟，没有用户愿意选择下单后无法付款的操作，所以此方案排除。
3.  即现有方案，下单后，支付后库存扣减。这样做的好处是用户下单、付款整个流程畅通无阻。但是问题也很明显，就是有一个出库失败，需要进行退款操作，用户实际使用中付款后的订单，自动取消了，确实是一个不合理的行径。但是这样做的额外的好处就是可以对接二开的库存调配接口，方便用于erp的对接，如果库存不足，则用erp对库存进行调配，然后使得一个订单可以顺利进入到下一个步骤。（参考京东的出库失败，京东有一个强大的仓库系统进行调货等等，即可能送货到达时间比约定时间慢很多）

#### 订单类创建步骤关系图[](#订单类创建步骤关系图)

![](https://docs.pickmall.cn/architecture/images/trade-build-class.png)

> 通过TradeBuilder 统一调度CartRenderStep的实现类型，来实现购物车的各个流程的渲染。
> 
> 在这个地方，购物车价格计算和结算页面的价格计算是完全服用代码的，只需要在渲染购物车时，对步骤进行精简即可，在现有系统中，通过步骤声明对方法实现，代码如下
> 
> ```
>     
>     int[] defaultRender = {0, 1, 2, 4, 5, 6, 7};
> 
>     
>     int[] cartRender = {0, 1, 2, 5};
> 
> ```
> 
> 每个注入到购物车渲染的步骤都有@Order注解，用于处理顺序，这样在代码渲染购物车信息时，可以通过预先定义好要走的流程来决定渲染出来的结果包含什么方法，如图中TradeBuild类中两个方法，buildCart 即为渲染购物车，buildTrade 即为渲染正比交易。
> 
> 如果在实际使用过程中需要去除一部分代码，则可以通过控制这一块代码来处理。代码中的热插件不是没有想过，只是多次尝试架构后发现不是很优雅，所以最终选择了一个笨办法，如果有二开的同学可以继续优化完善这块，将逻辑适配到您的系统中