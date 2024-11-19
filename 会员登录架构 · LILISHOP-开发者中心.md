# 会员登录架构 · LILISHOP-开发者中心
会员登录架构[](#会员登录架构)
-----------------

### 会员账号密码登录[](#会员账号密码登录)

#### 关系图[](#关系图)

![](https://docs.pickmall.cn/architecture/images/tokenLoginClass.png)

##### 说明[](#说明)

> MemberBuyerController 用户登录api控制器
> 
> MemberService 会员服务
> 
> SmsUtil 短信验证码认证工具
> 
> VerificationService 滑块验证码验证工具
> 
> TokenUtil token生成工具
> 
> Token token对象

#### 流程图[](#流程图)

![](https://docs.pickmall.cn/architecture/images/token.jpg)

##### 账号密码登录[](#账号密码登录)

> 1、携带uuid与服务端交互，服务端记录唯一id，并且将第三方登录地址返回
> 
> 2、将第三方返回的信息在redis缓存中记录，
> 
> 3、返回accesstoken（访问token），以及refreshtoken（过期刷新token）
> 
> 4、请求api时携带accesstoken
> 
> 5、服务端校验时，将前端token带入到redis中进行判定，如果存在则认为是前端的正常请求，予以放行
> 
> ##### 刷新token[](#刷新token)
> 
> 6、携带过期accesstoken请求后段，
> 
> 7、api查验发现redis中没有token记录
> 
> 8、返回403状态
> 
> 9、前端判定到403状态，携带refreshtoken去请求刷新token接口
> 
> 10、判定redis中的refreshtoken是否有效
> 
> 11、根据refreshtoken中信息，从新构建accesstoken以及refreshtoken
> 
> 12、返回accesstoken（访问token），以及refreshtoken（过期刷新token）
> 
> 13、前端刷新当前页面，重新请求api
> 
> 14、正确响应

### 第三方登录[](#第三方登录)

#### 主关系图[](#主关系图)

![](https://docs.pickmall.cn/architecture/images/thirdLoginClass.png)

##### 说明[](#说明)

> ConnectBuyerController 联合登陆api控制器
> 
> ConnectBuyerBindController 联合登陆绑定api控制器
> 
> MiniProgramBuyerController 小程序联合登陆api控制器
> 
> MemberService 会员服务
> 
> ConnectUtil 联合登录工具
> 
> ConnectService 联合登陆Service
> 
> Cache 缓存工具

#### 联合登陆扩展关系图[](#联合登陆扩展关系图)

![](https://docs.pickmall.cn/architecture/images/connect-plus.png)

##### 说明[](#说明)

> 这里是联合登陆内部关系图，如果有二开需求，可以简单看一下这块架构
> 
> 1、ConnectUtil 是联合登陆的核心工具类，他负责整体流程走向，系统调用等等一切逻辑，是这块的入口
> 
> 2、AuthRequest 接口，里边声明了获取token、刷新token、联合登陆地址、获取用户信息等接口
> 
> 3、AuthDefaultRequest 是联合登陆的默认实现类，默认返回一般都是异常，告知功能未实现
> 
> 4、AuthXXRequest 是具体功能实现累，Wechat代表微信，QQ代表qq，WechatPC代表pc扫码等等，实际扩展也是扩展此类即可，里边包含功能等具体实现，继承AuthDefaultRequest，如果开发中不需要实现某个接口，不进行代码调整即可，调用时会返回AuthDefaultRequest中的默认信息。
> 
> 5、ConnectAuthEnum 联合登陆美剧，支持的第三方登录方式
> 
> 6、ConnectAuth 联合登陆返回的对象，包含openid ，头像等等参数

#### 流程图[](#流程图)

![](https://docs.pickmall.cn/architecture/images/thirdLogin.png)

##### H5登陆[](#h5登陆)

> 1、请求服务，携带uuid用户唯一标识
> 
> 2、服务端返回携带用户标识的登陆地址
> 
> 3、如果用户已登陆状态，则将用户id写入回调地址，以待后续的第三方登录绑定
> 
> 4、跳转第三方进行登陆
> 
> 5、如果已登陆用户，则信息中包含用户id，使第三方信息与平台用户绑定
> 
> 6、将解析的openid，uuid 以及可能传递的id字段回传给前台
> 
> 7、请求api进行联合登陆
> 
> 8、根据redis中记录的信息，使得用户与第三方绑定
> 
> 9、返回登陆成功

##### App登陆[](#app登陆)

> 10、uniapp 封装，请求获取openid
> 
> 11、将方法回传的openid记录
> 
> 12、openid与uuid加密后请求服务端，进行登陆/绑定
> 
> 13、根据redis中记录的信息，使得用户与第三方绑定
> 
> 14、返回登陆结果

##### 小程序登录[](#小程序登录)

> 15、uniapp 获取用户信息，请求微信
> 
> 16、获取用户信息记录
> 
> 17、携带所有信息（包含手机号）进行登陆请求，
> 
> 18、服务端会根据手机号判定是否有会员，如果有则绑定当前会员与微信关系；如果没有，则注册登录结果一气呵成。
> 
> 19、返回登陆结果