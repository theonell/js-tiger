这个我们来写一个前后端不分离的项目来演示。

```
mkdir session
cd session
```

初始化一个 nodejs 的项目

```
npm init -y
```

安装 express

```
npm i --save express
```

下面来写一个最简单 http 服务器

```js
const express = require('express');
const app = express();

app.listen(3006, function(){
  console.log('running on port 3006...');
})
```

下面来实现一个 API ，返回一个静态 HTML 页面


```diff
+ app.get('/', function(req, res){
+  res.sendFile('index.html', {root: 'public'});
+ })
```

然后添加 public/index.html 如下

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Document</title>
</head>
<body>
  <a href="/login">登录</a>
</body>
</html>
```

同时，package.json 中做如下修改

```diff
+   "start": "nodemon index.js"
```

这样，我后台运行

```
npm start
```


public/login.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Login</title>
</head>
<body>
  <form action="/login" method="post">
    <input type="text">
    <input type="submit">
  </form>
</body>
</html>
```

浏览器中打开

```
http://localhost:3006/
```

也能够看到 index.html 的内容，点链接也可以访问 login.html。


### 书写服务器对应接口

index.js 中修改如下：

```diff
app.use(express.static('public'));
+const bodyParser = require('body-parser');
+
+// parse application/x-www-form-urlencoded
+app.use(bodyParser.urlencoded({ extended: false }))

app.get('/', function(req, res){
  res.sendFile('index.html', {root: 'public'});
})

+app.post('/login', function(req, res){
+  console.log(req.body);
+})
+
app.listen(3006, function(){
```


### 重定向 redirect

index.js 做出如下修改

```diff
app.post('/login', function(req, res){
+  let username = req.body.username;
+  // User.find({username: username}) 如果数据库中能找到这个用户
+  // 同时密码也对，这样才算登录成功
   console.log(req.body);
+  if(true) {
+    res.redirect('/');
+  }
 })
```

上面 redirect 接口实现的是“页面重定向”，具体的效果就是，前端页面会自动跳转到指定页面，对应上面的情况，就是自动跳转到首页。


### 使用 req.session 接口

index.js 修改如下

```diff
const bodyParser = require('body-parser');

+const session = require('express-session')
+
+app.use(session({
+  secret: 'keyboard cat',
+  resave: false,
+  saveUninitialized: true
+}))
+
// parse application/x-www-form-urlencoded
app.use(bodyParser.urlencoded({ extended: false }))

@@ -12,6 +20,7 @@ app.get('/', function(req, res){

app.post('/login', function(req, res){
  let username = req.body.username;
+  req.session.username = username;
  // User.find({username: username}) 如果数据库中能找到这个用户
```

上面代码有了，就可以拥有一个特殊的变量了 `req.session` 保存到这个变量中的数据，可以在各个 API 之间（页面之间）共享。只要本次会话不结束，这个变量就不会消失。

### 观察浏览器中的现象

浏览器中，我们到 login 页面，填写用户名，提交，这样后端执行的是

```
app.post('/login')
```

那么里面会有 session 的操作，也就是

```
req.session.username = username
```

也就是，服务器端的 session 已经创建了。

那么浏览器中的体现就是有一个 cookie 被创建了，到 Application -> storeage -> Cookie -> localhost:3006 之下，就可以看到，有这样的 cookie 数据：

```
connect.sid   xxxxxfdsjfkldsjfklsdjxxxx
```

上面的 sid 意思就是 Session Id ，也就是是服务器端 req.session 的对应的 id  。由于，客户端，每次请求都会携带 cookie 去服务器端，所以后续每次请求，都可以拿到服务器端 req.session 中存储的数据。


例如：

```js
app.get('/hello', function(req, res){
  res.send(req.session.username)
  })
```

小陷阱：后台每次添加代码，服务器都会重启，所以要重新执行一下 POST /login 生成一下 session 才能测试。

### 使用 pug 模板

现在在 GET / 这个接口中，我们有了 req.session.username 也就是登录用户的用户名，但是现在我们想要在页面中显示这个变量，这个就不能直接用 index.html 了，而要使用一种模板语言。模板有多种，其中 [pug](https://pugjs.org/api/getting-started.html) 是常见的一种。

先来装包：

```
npm i --save pug
```


index.js 代码修改如下

```diff
const session = require('express-session')
+const pug = require('pug');
+app.set('view engine', 'pug');
+

app.use(session({
  secret: 'keyboard cat',
@@ -15,7 +18,8 @@ app.use(bodyParser.urlencoded({ extended: false }))

app.get('/', function(req, res){
  console.log('home page', req.session.username);
-  res.sendFile('index.html', {root: 'public'});
+  let currentUser = req.session.username;
+  res.render('index', { username: currentUser })
})
```

删除 public/index.html ，新增 views/index.pug 文件

```
html
  body
    p= '当前用户名是：'
    h1= username
```


这样，再次登录一下，跳转到首页，就能显示出当前用户名了。

### 添加 logout 功能

index.js 中

```diff
const session = require('express-session')
+const pug = require('pug');
+app.set('view engine', 'pug')
+

app.use(session({
  secret: 'keyboard cat',
@@ -15,13 +18,18 @@ app.use(bodyParser.urlencoded({ extended: false }))

app.get('/', function(req, res){
  console.log('home page', req.session.username);
-  res.sendFile('index.html', {root: 'public'});
+  let currentUser = req.session.username;
+  res.render('index', { currentUser })
})
app.get('/login', function(req, res){
  res.sendFile('login.html', {root: 'public'});
})

+app.get('/logout', function(req, res){
+  req.session.destroy();
+  res.redirect('/');
+})
app.post('/login', function(req, res){
  let username = req.body.username;
```

views/index.pug 的功能为：

```
html
  body
    if currentUser
      span= currentUser
      a(href='/logout') 退出
    else
      a(href='/login') 登录
```

### 单页面应用中 Cookie 使用不方便了

因为涉及到跨域问题。

在 React-Axios 环境下，前端和后端代码，其实两个独立的项目，跑在不同的域名之上。导致了收发 cookie 默认都是不允许的，所以 req.session 本身也是不工作的，使用意义不大了。

但是后续，我们采用的主要架构，还是前后端分离架构，req.session 虽然不能用，但是我们会采用非常类似的原理来实现登录态和购物车等效果。


### 参考文档

- [codeforgeek](https://codeforgeek.com/2014/09/manage-session-using-node-js-express-4/)

- [Everything You Ever Wanted To Know About Authentication in Node.js]( https://www.youtube.com/watch?v=yvviEA1pOXw&t=128s)


### 本节全部代码

views/index.pug

```js
html
  body
    if currentUser
      p= currentUser
      a(href='/logout') 退出
    else
      a(href='/login') 登录
```

index.js

```js
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.urlencoded({ extended: false }));

const session = require('express-session')
const pug = require('pug');

app.set('view engine', 'pug');

app.use(session({
  secret: 'keyboard cat',
  resave: false,
  saveUninitialized: true
}))


app.get('/', function(req, res){
  let currentUser = req.session.username;
  res.render('index', {currentUser});
})

app.get('/login', function(req, res){
  res.sendFile('login.html', {root: 'public'});
})

app.post('/login', function(req, res){
  let username = req.body.username;
  req.session.username = username;
  res.redirect('/');
})

app.get('/logout', function(req, res){
  req.session.destroy();
  res.redirect('/');
})

app.listen(3006, function(){
  console.log('running on port 3006...');
})
```

public/login.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Document</title>
</head>
<body>
   <form action="/login" method="post">
     用户名：
     <input name="username" type="text">
     <input type="submit">
   </form>
</body>
</html>
```
