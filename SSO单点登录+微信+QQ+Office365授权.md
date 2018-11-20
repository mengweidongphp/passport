## SSO单点登录+微信+QQ+Office365授权
**单点登录的解释**
单点登录（Single Sign On），简称为 SSO，是目前比较流行的企业业务整合的解决方案之一。SSO的定义是在多个应用系统中，用户只需要登录一次就可以访问所有相互信任的应用系统。

**实现的方法**
**server端**

 - “共享Cookie”即共享session的方式,本质上cookie只是存储session-id的介质,session-id也可以放在每次请求的url里面.session机制是一个server一个session。（通过共享cookie，共享session）

 - SSO-Token方式是因为共享session的方式不安全,所以我们不再以session-id作为身份的标识,我们另外生成一种标识,把它取名为SSO-Token,这种标识在整个server群唯一的,所以所有的server群都能验证整个token,同时拿到token，就代表拿到用户的信息（**这里我们实现这一种**）

 - 几张图片能更好的理解单点登录![在这里插入图片描述](https://img-blog.csdnimg.cn/20181116180335386.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2NDk1MDc3,size_16,color_FFFFFF,t_70)
 -![在这里插入图片描述](https://img-blog.csdnimg.cn/20181116181000466.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2NDk1MDc3,size_16,color_FFFFFF,t_70)

**技术实现的机制**
当用户第一次访问应用系统的时候，因为还没有登录，会被 
引导到认证系统中进行登录；根据用户提供的登录信息，认证系统进行身份校验，如果通过校验，应该返回给用户一个认证的凭据－－ticket；用户再访问别的应用的时候，就会将这个ticket带上，作为自己认证的凭据，应用系统接受到请求之后会把ticket送到认证系统进行校验，检查ticket的合法性。如果通过校验，用户就可以在不用再次登录的情况下访问应用系统2和应用系统3了。 
要实现SSO，需要以下主要的功能： 
所有应用系统共享一个身份认证系统。

 - 统一的认证系统是SSO的前提之一。认证系统的主要功能是将用户的登录信息和用户信息库相比较，对用户进行登录认证；认证成功后，认证系统应该生成统一的认证标志（ticket），返还给用户。另外，认证系统还应该对ticket进行效验，判断其有效性。
   所有应用系统能够识别和提取ticket信息
 - 要实现SSO的功能，让用户只登录一次，就必须让应用系统能够识别已经登录过的用户。应用系统应该能对ticket进行识别和提取，通过与认证系统的通讯，能自动判断当前用户是否登录过，（这一步目前还没涉及）

***这之前是对单点登录的理解***
**一.先做第三方授权**
提前准备

 1. laravel5.6以上版本框架，基于laravel框架，因为有些包是通过laravel框架composer引入
 2. 申请通过的qq互联账号，具备开发资质，qq互联地址：[https://connect.qq.com/index.html](https://connect.qq.com/index.html)
 3. 申请通过的微信开放平台账号，申请一个应用。
 4. 在office356应用程序注册门户中注册Web应用程序

1.微信和qq授权
> 有两篇比较好的文档

 - 文档一（laravel官方的文档,跟着文档就能撸出来）
 文档链接:[https://laravelacademy.org/post/1321.html](https://laravelacademy.org/post/1321.html)
 
 - 文档二
 文档链接:[https://www.cnblogs.com/zzdylan/p/5922477.html](https://www.cnblogs.com/zzdylan/p/5922477.html)
 
2.office365授权
 - 文档链接:[https://github.com/microsoftgraph/msgraph-training-phpapp/blob/master/Lab.md](https://github.com/microsoftgraph/msgraph-training-phpapp/blob/master/Lab.md)
 - 授权类操作文档:[https://packagist.org/packages/league/oauth2-facebook](https://packagist.org/packages/league/oauth2-facebook)

**二.授权登录完成后，实现单点登录**

> 这里我们模仿Oauth2.0自己写一个类似的。laravel自身有passport来着OAuth2.0授权，是比较好的选择，自己还没研究（后期研究完会更新上来）。

*流程图*
![在这里插入图片描述](https://img-blog.csdnimg.cn/2018111915432995.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2NDk1MDc3,size_16,color_FFFFFF,t_70)
1.准备单点登录表
```
DROP TABLE IF EXISTS `mouth_oauth`;
CREATE TABLE `mouth_oauth` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `redirect_uri` varchar(500) NOT NULL COMMENT '网站域名',
  `mouth_appid` varchar(500) NOT NULL DEFAULT '' COMMENT 'addid',
  `public_key` varchar(500) NOT NULL DEFAULT '' COMMENT '公钥',
  `private_key` varchar(500) CHARACTER SET utf8 COLLATE utf8_unicode_ci DEFAULT NULL COMMENT '私钥',
  `created_at` datetime DEFAULT NULL COMMENT '记录创建时间',
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '记录更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8 COMMENT='单点登录oauth表';
```
单点登录表放到主站里，每个子站对应一个appid，每个子站请求的时候通过本地配置文件拿到公钥和私钥，通过自己的加密方式后得到secret，和数据库相对应的加密后的secret相比较是否一致。一致可以请求到该网站登录入口，不一致请求不到。

2.大概流程
比如说:网站A域名为:www.aa.com
			单点登录域名为:www.passport.com
			主站域名为:www.master.com

> 主站和单点登录页面公用一个数据库，两个服务器，两个不同域名

<hr>
当访问www.aa.com的时候，会走中间件从配置文件里读取appid和公钥，私钥，经过自己算法加密以后行成secret，跳转到单点登录域名页面。

> 例如：www.passport.com/middleware?secret=XXXX&appid=XXXX

**a.跳转到单点登录页面**
跳转到单点域名www.passport.com以后拿到appid，通过appid从数据表mouth_oauth里查询该条数据的公钥和私钥*，同时把这条数据储存到session里，留到第三方授权的时候使用，因为授权以后无法带参数，说要储存到session里。*通过同样算法加密以后行成secret，然后和传递过来的secret相对比。如果相同说明该访问可以正常跳转到www.passport.com登录页面，不相同说明不可以访问，侧返回。

**b.当成功到达单点登录域名:www.passport.com以后，要执行登录流程。**
	1.账号登录或第三方授权登录
	**账号登录**:输入账号和密码执行登录，从主站数据库检索该账号和密码，正确以后把该网站配置文件里的公钥和私钥和id通过算法拼接成secret，ID也传过去，重定向到该数据的redirect_uri域名。
	**第三方授权登录**:原理同账号登录，只不过走第三方授权以后没有办法从url里拿到参数，前面说到存储到session，当授权拿到用户信息以后，再拼接secret拼接对比，查询该信息是否绑定过，绑定过的话直接登录，没有绑定过得话跳转到登录页面，执行绑定。


> 	例如:www.aa.com/login?secret=XXXX&id=1;

 2.passport判断用户登录后，跳转到 www.aa.com
 拿到参数secret和id，用传过来的id和本网站配置文件里的私钥和公钥加密后形成secret。和传递过来的secret相对比，相同说明可信赖，执行传递过来的id登录该网站，实现了www.aa.com网站的登录。
 
 





 


 

