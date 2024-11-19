# 系统日志架构 · LILISHOP-开发者中心
系统日志架构[](#系统日志架构)
-----------------

#### 类关系图[](#类关系图)

![](https://docs.pickmall.cn/architecture/images/systemLog-class.png)

##### 说明[](#说明)

> 1.  SystemLogPoint 注解切入点，描述信息description，详细操作内容customerLog，支持Spel语法（即类似：#order.sn）
>     
> 2.  SystemLogService 系统日志业务类，全平台任意serviceImpl中加入注解即可，会自动根据会员信息存储日志，使用示例如下：
>     
>     ```
>         @SystemLogPoint(description = "修改订单", customerLog = "'订单[' + #orderSn + ']收货信息修改，修改为'+#memberAddressDTO.consigneeDetail+'")
>         public Order updateConsignee(String orderSn, MemberAddressDTO memberAddressDTO) {
>             Order order = OperationalJudgment.judgment(this.getBySn(orderSn));
>     
>             String message = "订单[" + orderSn + "]收货信息修改，由[" + order.getConsigneeDetail() + "]修改为[" + memberAddressDTO.getConsigneeDetail() + "]";
>     
>             BeanUtil.copyProperties(memberAddressDTO, order);
>             this.updateById(order);
>     
>             OrderLog orderLog = new OrderLog(orderSn, UserContext.getCurrentUser().getId(), UserContext.getCurrentUser().getRole().getRole(), UserContext.getCurrentUser().getUsername(), message);
>             orderLogService.save(orderLog);
>     
>             return order;
>         }
>     
>     ```
>     
> 3.  SystemLogAspect 切入点拦截类，对请求处理时间，请求者ip，请求对象，请求用户名，请求api，请求参数等等做了限制：潜质通知即为记录当前操作的时间，切入点进行日志的记录处理
>     
>     ```
>     
>         
>         @Pointcut("@annotation(cn.lili.common.aop.syslog.annotation.SystemLogPoint)")
>         public void controllerAspect() {
>     
>         }
>     
>         
>         @Before("controllerAspect()")
>         public void doBefore() {
>             beginTimeThreadLocal.set(new Date());
>         }
>     
>     ```
>     
> 4.  日志对象内容的填充，这里根据spel对对象的日志内容进行数据填充。在操作人记录时，对从token中解析的用户进行处理，将操作对象进行填充，根据角色类型，用户id，用户名称，在后期日志查看时可以精准定位操作人员。
>     
>     ```
>      
>         @AfterReturning(returning = "rvt", pointcut = "controllerAspect()")
>         public void after(JoinPoint joinPoint, Object rvt) {
>             try {
>                 Map map = spelFormat(joinPoint, rvt);
>                 String description = map.get("description").toString();
>                 String customerLog = map.get("customerLog").toString();
>     
>                 Map<String, String[]> logParams = request.getParameterMap();
>                 AuthUser authUser = UserContext.getCurrentUser();
>                 SystemLogVO systemLogVO = new SystemLogVO();
>     
>                 if (authUser == null) {
>                     
>                     systemLogVO.setStoreId(-2L);
>                     
>                     systemLogVO.setUsername("游客");
>                 } else {
>                     
>                     systemLogVO.setStoreId(authUser.getRole().equals(UserEnums.STORE) ? Long.parseLong(authUser.getStoreId()) : -1);
>                     
>                     systemLogVO.setUsername(authUser.getUsername());
>                 }
>     
>                 
>                 systemLogVO.setName(description);
>                 
>                 systemLogVO.setRequestUrl(request.getRequestURI());
>                 
>                 systemLogVO.setRequestType(request.getMethod());
>                 
>                 systemLogVO.setMapToParams(logParams);
>                 
>     
>                 
>                 systemLogVO.setIp(IpUtils.getIpAddress(request));
>                 
>                 systemLogVO.setIpInfo(ipHelper.getIpCity(request));
>                 
>                 systemLogVO.setCustomerLog(customerLog);
>                 
>                 long beginTime = beginTimeThreadLocal.get().getTime();
>                 long endTime = System.currentTimeMillis();
>                 
>                 Long usedTime = endTime - beginTime;
>                 systemLogVO.setCostTime(usedTime.intValue());
>                 
>                 ThreadPoolUtil.getPool().execute(new SaveSystemLogThread(systemLogVO, systemLogService));
>     
>             } catch (Exception e) {
>                 log.error("系统日志保存异常", e);
>             }
>         }
>     
>     ```
>     
> 
> 1.  在日志保存，系统会另起一个线程进行处理，保证订单业务响应不受影响
>     
>     ```
>         
>         private static class SaveSystemLogThread implements Runnable {
>     
>             private final SystemLogVO systemLogVO;
>             private final SystemLogService systemLogService;
>     
>             public SaveSystemLogThread(SystemLogVO systemLogVO, SystemLogService systemLogService) {
>                 this.systemLogVO = systemLogVO;
>                 this.systemLogService = systemLogService;
>             }
>     
>             @Override
>             public void run() {
>                 try {
>                     systemLogService.saveLog(systemLogVO);
>                 } catch (Exception e) {
>                     log.error("系统日志保存异常,内容{}：", systemLogVO, e);
>                 }
>             }
>         }
>     
>     ```
>     

##### 扩展说明[](#扩展说明)

> 这是本系统推荐的一种日志机制，保证前端响应速度的同时，使得日志对整体架构的影响缩小到最小，在日后的其他文档中也会经常看到类似的文档。这里如果资源充沛，其实也可以使用MQ去实现。

#### 流程图[](#流程图)

![](https://docs.pickmall.cn/architecture/images/log-flow.png)

##### 日志流程描述[](#日志流程描述)

> 1、业务类操作，可能是系统，也可能是用户，在日志中会有对应的识别
> 
> 2、业务类执行
> 
> 4、切入点被执行，通常情况下执行与对应方法执行后，因为如果在其他位置执行，可能影响系统的正常流程。解析Spel语法，即类似_#order.sn_ 这样的语法
> 
> 5、线程池获取线程，传入要保存的数据
> 
> 6、线程异步保存日志