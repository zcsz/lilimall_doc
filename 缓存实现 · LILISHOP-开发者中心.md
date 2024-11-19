# 缓存实现 · LILISHOP-开发者中心
@Cacheable 使用方案[](#cacheable-使用方案)
----------------------------------

#### 缓存配置[](#缓存配置)

```
lili:
# 使用Spring @Cacheable注解失效时间
  cache:
    # 过期时间 单位秒 永久不过期设为-1
    timeout: 300

```

#### 使用方法[](#使用方法)

1.  代码使用了mybatisplus ，所以很多代码都在架构的封装里，这里表明使用方法

###### 业务接口代码中 XXXXService.java

```
   //定义缓存key前缀
   @CacheConfig(cacheNames = "bill")
   public interface IBillService extends IService<Bill> {
   //重写接口中的根据id获取对象方法
           @Override
       @Cacheable(key = "'bill_'+#id")
       Bill getById(Serializable id);

   //重写接口中的修改对象方法 清除之前获取对象时的缓存
           @Override
       @CacheEvict(key = "'bill_'+#bill.id")
       Bill saveOrUpdate(Bill bill);

   }

```

#### 实现效果[](#实现效果)

> ttl key
> 
> \-1 代表永久有效
> 
> \-2 代表已过期
> 
> 其他正整数，代表剩余秒

###### 查询

还有298秒过期

![](https://docs.pickmall.cn/development/images/image-20200619154137378.png)

###### 修改

已过期

![](https://docs.pickmall.cn/development/images/image-20200619154033908.png)