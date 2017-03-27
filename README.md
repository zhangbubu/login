> 项目介绍

* 采用`vue-cli` 构建初始项目目录
* 使用`vue2`、`vue-router2`、`Element-ui`、`express`、`webpack2`、`gulp`构建项目开发环境
* 前端采用Vue、Element-ui搭建页面以及Vuex进行数据处理，后端采用express完成注册、登录、登出、登录状态查询的restful API
* 页面为注册页，登录页，主页

> 目录及文件的改动，均含详细备注

1. package.json中的'dependencies'包含的是增加的npm包
2. config/index.js 中设置了请求端口代理，能够跨域访问服务器端口（8080 => 3000）
3. server  主要是服务端的路由以，api接口，mongodb的配置操作
4. src 为vue应用的主要文件，包含路由，component...
5. gulpfile.js 增加服务器重启和浏览器刷新任务
6. server.js  服务端启动文件


> 操作指令

```shell
npm install 安装依赖

npm run dev 使用webpack开启前端资源的打包编译

npm run server  启动服务端并开启浏览器

```

这里需要双开两个命令行窗口，一个负责前端的编译，一个负责服务端的任务流

> 环境搭建的详细解决思路

[前端热更新，后端服务重启，浏览器自动刷新]( http://selvinpro.com/2017/03/20/browser-reload/#more)

- [x] 实现了基本登录 cookie && session
* [ ] 独立系统登录 token
* [ ] 第三方登录 access_token






#### 详谈注册/登录/登出

> 登录功能实现的原理

浏览器用户提交表单后与服务器产生会话，传递给服务器一个识别信息，同时服务器这边有相应的数据，因此能够对其进行比较，然后识别是否为正确的用户，而且一般需要设定此数据的存储时长，也就是用户认证的时长，过了这个时间段，则会失效，需要重新登录，因此这里需要有四个关键点，**浏览器的信息**，**服务器的信息**，**两个信息的对比**，**时效设定**。

> 登录功能实现的几种方式

1. [基本登录 cookie && session](#1)

   浏览器提交用户表单后，服务器接受表单中的用户信息，将其存在session表中（session表存在内存，数据库，缓存等），然后以cookie的方式（cookie中存有sessionId及对应值）传回给浏览器，这样只要浏览器的cookie没有失效，则在这段时间内，服务端便能正确识别用户信息，不用重复登录

   * 服务器端的产生Session ID
   * 服务器端和客户端存储Session ID
   * 从HTTP Header中提取Session ID
   * 根据Session ID从服务器端的Hash中获取请求者身份信息

2. `token`（JSON Web Token）

   ​

3. 第三方：`access_token`


<h4 id="1">基本登录 cookie && session </h4>

1. 通过mongoose建立数据模型，以name为主索引并设置其唯一属性，此例中，我们服务端采用内存储存sessionId，也可尝试mongoDB或redis等，均有相应的中间件可以使用。

2. 登录   POST：  `/api/login` 
  在前端vue中怎样识别登录状态，当输入用户表单后点击登录，我们向`/api/login `发起了`pos`t请求，`/api/login `对应的控制器中进行了这样几步流程

  * 将request中包含的用户信息（用户名，密码），生成mogoose实例

  * 进入mongoDB中查找此用户名的信息（User 的Skema模型中已经设定name为唯一的索引，不能重复的）

  * 没有找到，返回无此用户名，找到则与数据库中的密码与此密码进行比较（在生成mongoose实例时，都将密码进行了sha1加密）

  * 密码匹配正确，返回正确信息，并写入session

        // 用户信息写入 session
        delete user.password;
        req.session.user = user;

  * 密码匹配失败，否则返回密码错误的信息

    在其它如/api/register、/api/user中通过判断req.session.user的存在与否，来实现登录登出状态的改变

  ```javascript
  checkLogin(req, res, next) {
      if (!req.session.user) {
        return res.json({
           error: '未登录'
        });
      }
      next();
    },

    checkNotLogin(req, res, next) {
      if (req.session.user) {
        return res.json({
          error: '已登录'
        });
      }
      next();
    }
  ```

  由此，当流程跑通之后，我们能够得到返回登录成功的信息，接下来就可以在前端通过`vue-router`实现跳转，以及将获取的user信息填入`localstorage`，供其它页面使用其数据。

3. 注册   POST： `/api/register`

   注册功能的实现，同样当我们填写完注册表单后， `/api/register`对应的控制器中将接受我们提交之后的数据。

   * 通过request获取用户名和密码，并将密码加密（此处采用sha1），生成mongoose实例，同时通过`objectid-to-timestamp `将实例中的objectId转化为时间格式，即为此实例的创建时间，具体可看**疑问详解**。
   * 通过唯一的name属性，进入数据库查询，看该用户名是否已经被注册，返回相应的数据。

4. 登出    GET：`/api/user`


   此api的作用是实现登出功能的，当点击登出后，vue通过axios访问此api，api对应的控制器中将执行以下代码

   ```javascript
   req.session.user = null;
   res.json({
     message:'登出成功'
   })
   ```



> 疑问详解

1. objectid-to-timestamp 包的作用

   首先了解MongoDB中的`ObjectId`，每一个document（即MySQL中的row每一行数据），均含有一个`_id`的属性，而其属性值即为`ObjectId`

   `ObjectId` 是一个12字节 BSON 类型数据，有以下格式：

   - 前4个字节表示时间戳（**数据创建的时间**）
   - 接下来的3个字节是机器标识码
   - 紧接的两个字节由进程id组成（PID）
   - 最后三个字节是随机数。

   因此它的作用即是将前四个字节转化为时间格式。

2. ​
