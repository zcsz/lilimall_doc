# 流量限制架构 · LILISHOP-开发者中心
流量限制架构[](#流量限制架构)
-----------------

#### 示例代码[](#示例代码)

```
    
    @LimitPoint(name = "sms_send", key = "sms")
    @ApiImplicitParams({
            @ApiImplicitParam(paramType = "path", dataType = "String", name = "mobile", value = "手机号"),
            @ApiImplicitParam(paramType = "header", dataType = "String", name = "uuid", value = "uuid"),
    })
    @GetMapping("/{verificationEnums}/{mobile}")
    @ApiOperation(value = "发送短信验证码")
    public ResultMessage getSmsCode(
            @RequestHeader String uuid,
            @PathVariable String mobile,
            @PathVariable VerificationEnums verificationEnums) {
        if (verificationService.check(uuid, verificationEnums)) {
            smsUtil.sendSmsCode(mobile, verificationEnums, uuid);
            return ResultUtil.success(ResultCode.VERIFICATION_SEND_SUCCESS);
        } else {
            return ResultUtil.error(ResultCode.VERIFICATION_SMS_EXPIRED_ERROR);
        }
    }

```

#### 使用方法[](#使用方法)

1.  @Limit 限流注解，提供的参数有：
    
    ```
    
    @Target({ElementType.METHOD, ElementType.TYPE})
    @Retention(RetentionPolicy.RUNTIME)
    @Inherited
    @Documented
    public @interface LimitPoint {
        
        String name() default "";
    
        
        String key() default "";
    
        
        String prefix() default "";
    
        
        int period() default 60;
    
        
        int limit() default 10;
    
        
        LimitType limitType() default LimitType.IP;
    }
    
    ```
    

#### 实现效果[](#实现效果)

![](https://docs.pickmall.cn/architecture/images/image-20200616122357697.png)

#### 使用技巧[](#使用技巧)

可以用在限流接口、短信验证码接口、限时抢购接口，防止接口被攻击。一般短信验证码比如60秒一次，那么这边限制50秒或者55秒配置。