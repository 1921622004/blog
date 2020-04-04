## 简介
**Express**是Node.js 的web开发框架，很受欢迎，这篇文章主要从源码来分析 **express**的路由以及中间件是如何初始化及工作的。
> 下面来先看下用法
```js
// 示例
const express = require('express');
const app = express();
// fn1
app.use((req, res, next) => {
    console.log('enter');
    next()
});
// fn2
app.get('/user', (req, res, next) => {
    console.log('user:app get 1.1');
    next()
}, (req, res, next) => {
    // fn3
    console.log('user:app get 1.2');
    next()
});
// fn4
app.get('/api', (req, res, next) => {
    console.log('api: app get 1');
    res.send('api');
});
// fn5
app.get('/user', (req, res, next) => {
    console.log('user: app get 2.1');
    next();
}, (req, res, next) => {
    // fn6
    console.log('user: app get 2.2');
    res.send('user');
});
app.listen(3000);
```
当我们在浏览器访问http://localhost:3000/user 的时候，看下输出了什么

下面一起来看下源码，研究一下其中的原理究竟如何

### **初始化阶段**



#### 首先先看下app.get  --  [源码位置](https://github.com/expressjs/express/blob/master/lib/application.js#L472)
```js
// 1.生成router实例
methods.forEach(function(method){
  app[method] = function(path){
     // 这个函数就是判断一下当前有没有router属性， 没有就给this添加一个router属性 属性值为 Router的一个实例
    this.lazyrouter();
    // 2. 此处method为get 也就是Router类的get方法 
    //    另外 path 就是当前的/user 
    //    slice就是Array.prototype.slice  所以后面的这个参数就是我们传递进去的处理函数
    var route = this._router.route(path);
    // 3. Router.route  返回了一个Route实例，同时生成了一个Layer并将layer添加到当前stack中，layer.path = path, layer.handler = route.dispatch方法 
    // 4. 然后调用之前返回的Route实例上的get 方法
    // 5. 然后再看看之前layer的handler 也就是route.dispatch 方法  这个方法就是逐个调用layer的启动方法，稍后会提到
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});
```
> Router.route 方法 -- [源码位置](https://github.com/expressjs/express/blob/master/lib/router/index.js#L491)
```js
Router.route = function route(path) {
  // 3.1 生成Route实例 route初始化stack
  // 3.2 同时route.methods = {}; 为了后期在匹配路径的时候提升效率
  var route = new Route(path);
  var layer = new Layer(path, {
    sensitive: this.caseSensitive,
    strict: this.strict,
    end: true
  }, route.dispatch.bind(route));
  layer.route = route;
  this.stack.push(layer);
  return route;
};
```

Route.get方法
```js
methods.forEach(function(method){
  // 4.1 同样methods遍历绑定方法
  Route.prototype[method] = function(){
    // handles 就是之前传递进来的handles method就是get
    var handles = flatten(slice.call(arguments));
    for (var i = 0; i < handles.length; i++) {
      var handle = handles[i];
      if (typeof handle !== 'function') {
        var type = toString.call(handle);
        var msg = 'Route.' + method + '() requires a callback function but got a ' + type
        throw new Error(msg);
      }
      debug('%s %o', method, this.path)
      // 遍历handles 每次遍历生成一个layer  layer.path = '/' layer.handler = 当前的路由函数
      var layer = Layer('/', {}, handle);
      layer.method = method;
      // route.methods['get'] = true;
      this.methods[method] = true;
      // 将layer推入当前stack
      this.stack.push(layer);
    }
    return this;
  };
});
```

#### app.use
> app.use 方法 注册中间件

```js
app.use = function use(fn) {
  var offset = 0;
  var path = '/';

  // 1.处理参数  初始化router
  this.lazyrouter();
  var router = this._router;
  fns.forEach(function (fn) {
    // 2.调用的是router.use 方法 path 默认 '/' fn
    if (!fn || !fn.handle || !fn.set) {
      return router.use(path, fn);
    }
    debug('.use app under %s', path);
    fn.mountpath = path;
    fn.parent = this;

    router.use(path, function mounted_app(req, res, next) {
      var orig = req.app;
      fn.handle(req, res, function (err) {
        setPrototypeOf(req, orig.request)
        setPrototypeOf(res, orig.response)
        next(err);
      });
    });

    // mounted an app
    fn.emit('mount', this);
  }, this);

  return this;
};
```

router.use 方法

```js
proto.use = function use(fn) {
  var offset = 0;
  var path = '/';
  // 2.1 处理参数 
  if (typeof fn !== 'function') {
    var arg = fn;
    while (Array.isArray(arg) && arg.length !== 0) {
      arg = arg[0];
    }
    if (typeof arg !== 'function') {
      offset = 1;
      path = fn;
    }
  }
  var callbacks = flatten(slice.call(arguments, offset));
  if (callbacks.length === 0) {
    throw new TypeError('Router.use() requires a middleware function')
  }
  // 2.2 遍历中间件 向this.stack 加入layer layer.path = path layer.handle 就是中间件函数  layer.route = undefined
  for (var i = 0; i < callbacks.length; i++) {
    var fn = callbacks[i];

    var layer = new Layer(path, {
      sensitive: this.caseSensitive,
      strict: false,
      end: false
    }, fn);

    layer.route = undefined;

    this.stack.push(layer);
  }

  return this;
};
```

#### 初始化阶段总结
- **app._router** 每次调用get或者post等方法都会生成一个对应path的route，同时，生成一个layer 这个layer上面的handle方法是遍历对应这个route的所有方法的，其中的next方法就是调用 **router.stack** 中下一个layer的方法。
- 然后调用route的get或者post方法，*route*上的stack继续挂载一个layer 的stack，这里layer的handle方法就是对应传递进来的处理请求的方法，同时next就是调用当前 *route.stack* 中下一个layer的方法。
- **app.use**  调用router.use方法，不生成route，layer上的handle方法是调用**router.stack**中下一个layer的方法。
- 可能文字叙述有点不太清楚，看下图片。


![](https://user-gold-cdn.xitu.io/2019/4/17/16a2b93efd402457?w=1914&h=910&f=png&s=86825)

### 处理请求阶段
app.listen方法  
调用的是app.handle方法
```js
app.handle = function handle(req, res, callback) {
  var router = this._router;
  var done = callback || finalhandler(req, res, {
    env: this.get('env'),
    onerror: logerror.bind(this)
  });

  if (!router) {
    debug('no routes defined on app');
    done();
    return;
  }
// 这里的done就是一个错误控制函数
  router.handle(req, res, done);
};
```

然后看看router.handle做了什么，这里代码太多，稍稍修改了一下，[源码在这里](https://github.com/expressjs/express/blob/master/lib/router/index.js#L136)
```js
proto.handle = function handle(req, res, out) {
  var self = this;

  var idx = 0;
  var protohost = getProtohost(req.url) || ''
  var removed = '';
  var paramcalled = {};

  var stack = self.stack;

  var parentParams = req.params;
  var parentUrl = req.baseUrl || '';
  var done = restore(out, req, 'baseUrl', 'next', 'params');

  // setup basic req values
  req.baseUrl = parentUrl;
  req.originalUrl = req.originalUrl || req.url;

  next();

  function next(err) {
    // 错误处理 调用next('error')的时候
    var layerError = err === 'route' ? null : err;
    // 二级路由路径处理
    if (removed.length !== 0) {
      req.baseUrl = parentUrl;
      req.url = protohost + removed + req.url.substr(protohost.length);
      removed = '';
    }
    // 跳到下一个路由
    if (layerError === 'router') {
      setImmediate(done, null)
      return
    }
    // 遍历完当前layer，然后调用done方法，就是调用上一层router.stack的下一个函数的方法
    if (idx >= stack.length) {
      setImmediate(done, layerError);
      return;
    }
    var path = getPathname(req);
    if (path == null) {
      return done(layerError);
    }

    var layer;
    var match;
    var route;
    while (match !== true && idx < stack.length) {
      layer = stack[idx++];
      match = matchLayer(layer, path);
      route = layer.route;
      if (match !== true) continue
      if (!route) continue
      if (layerError) {
        match = false;
        continue;
      }
      var method = req.method;
      // 通过route的methods方法判断是否有get或post方法
      var has_method = route._handles_method(method);
      if (!has_method && method !== 'HEAD') {
        match = false;
        continue;
      }
    }
    if (match !== true) {
      return done(layerError);
    }
    // 如果layer.route存在 就放在req.route上 
    if (route) {
      req.route = route;
    }
    // 总之这里就是调用layer.handle_request
    // 记住这里的next是调用router.stack里的下一个路由或者中间件函数
    req.params = self.mergeParams
      ? mergeParams(layer.params, parentParams)
      : layer.params;
    var layerPath = layer.path;
    if(!layer.keys || layer.keys.length === 0 ) {
      if(route){
        layer.handle_request(req, res, next)
      } else {
        removed = layer.path;
        req.url = req.url.slice(0,removed.length);
        if(req.url == ''){
          req.url = '/';
          slashAdded = true;
        }
        layer.handle_request(req,res,next)
      }
    }
  }
};
```

然后再看layer.handle_request方法
```js
Layer.prototype.handle_request = function handle(req, res, next) {
  var fn = this.handle;
  if (fn.length > 3) {
    // 函数的length就是函数形参的长度
    return next();
  }
  fn(req, res, next);
};
```
调用layer.handle 还记得之前讲的吗，这里如果是使用app.use ，这里的handle就是传递进来的中间件函数，如果是app.get那就是调用的route.dispatch方法，就是让之前app.get传递进去的函数逐个执行。

Route.dispatch方法
```js
// 这里的done就是router.stack 中的next函数了。
Route.prototype.dispatch = function dispatch(req, res, done) {
  var idx = 0;
  var stack = this.stack;
  if (stack.length === 0) {
    return done();
  }
  var method = req.method.toLowerCase();
  if (method === 'head' && !this.methods['head']) {
    method = 'get';
  }
  req.route = this;
  next();
  function next(err) {
    if (err && err === 'route') {
      return done();
    }
    if (err && err === 'router') {
      return done(err)
    }
    var layer = stack[idx++];
    if (!layer) {
      return done(err);
    }
    if (layer.method && layer.method !== method) {
      return next(err);
    }
    if (err) {
      layer.handle_error(err, req, res, next);
    } else {
      layer.handle_request(req, res, next);
    }
  }
};
```

### 处理请求阶段总结
> 总结起来就是一张图。。。。。请一定原谅我的拙略画技好吗

![](https://user-gold-cdn.xitu.io/2019/4/17/16a2b9424132a67b?w=1914&h=910&f=png&s=86825)

### 感谢
以上，就是这篇文章的所有内容了，主要也就是讲解一下express路由的原理，当然，express.Router 用的也是 proto对象，理解了这些，Router的道理也就自然就懂了，后续会将express的路由匹配原理也整理成一篇文章发出来，感谢各位看官。
