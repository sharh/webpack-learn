# `webpack 4`的`SplitChunksPlugin`笔记

> `webpack 4`开始，原先的`webpack.optimize.commonschunkplugin`被替换为`SplitChunksPlugin`，并在`webpack.config.js`中以`optimization`字段来进行配置


> ``说明

```json

Optimization: {
  //是否压缩chunk，默认在production模式下是true
  minimize: true, 
  //提供一个自定义的代码压缩器，默认是UglifyjsWebpackPlugin
  minimizer: UglifyjsWebpackPlugin | [UglifyjsWebpackPlugin], 
  //运行时部分，只能提供名字，默认是false，每个chunk里面包含运行时。下面说明什么是runtimeChunk
  runtimeChunk: object|string|boolean, 
  //在chunk生成阶段不触发错误
  noEmitOnErrors: true,
  //是否让webpack设置process.env.NODE_ENV，如果给了字符串值，会使用webpack.DefinePlugin来定义全局变量，默认情况下使用的是webpack.config.js里配置的mode，如果没有配置mode的话则值是"production"
  nodeEnv: string|boolean,
  //代码分割，默认行为：
  //1、来自node_modules目录，或者被重复引用的模块
  //2、大于30kb 的chunk块
  //3、最大并行加载chunk数小于等于5，就是在多个模块被引用的次数
  //4、初始化页面最大并行加载小于等于3，初始化的那个地方需要下载的次数小于3
  //在满足最后2个条件的情况下，平衡文件大小，越大的越被分割。
  //大概意思应该是尽量减少请求数，请求的文件大小也不能太小
  splitChunks: {
    // 哪些类型的chunk，可以提供函数或者字符，字符的可选值为：
    // all, async, initial。如果是all表示可以在异步和同步的块中共享
    // function(chunk)直接提供了chunksFilter，可以过滤chunk。除了webpack.config.js中的entry申明的chunk有name，其他的chunk如果没有按webpack的加载设置chunkname，那他的name是null，webpack在没有配置cacheGroups的时候自动为其分配名称，如0/1/2等。
    chunks: "async",
    //最小的大小，超过这个大小的文件会被分割
    minSize: 30000,
    //最小包含的chunk数
    minChunks: 1,
    // 最大的异步请求数
    maxAsyncRequests: 5,
    //最大的初始化请求数
    maxInitialRequests: 3,
    //名称自动分割符
    automaticNameDelimiter: '~',
    // 名称，可以自定义，提供函数或者字符串
    name: true,
    //这个地方可以自定义分割
    cacheGroups: {
        // 分割的组名，这个只是组名，具体的分割名字是由其下面的name定义
        // 注意这个name对应的是output配置中的[name]属性
        commons: {
          //匹配规则，这里匹配node_modules目录下的
            test: /[\\/]node_modules[\\/]/,
            name: "commons",
            chunks: "all"
        },
        verdor: {
          //注意test可以是很多类型，正则表达式测试的是
          //module.nameForCondition()
          //string的话就直接看是否以string开头
          //function的话是test(module, module.getChunks())
          //不存在或者为true，表示都符合
            test: /bundle\.js$/,
            //不填name或者不存在的，直接会跳过，如果后面没有了，命名以key+automaticNameDelimiter+entryName命名
            name: "verdor",
            chunks: "all"
        },
        test: {
            test: /bundle1/,
            name: "test",
            chunks: "all"
        },
        //后面的权重在没有设置priority的话，越在下面的权重越高，设置了的话按priority的值来
        last: {
          //设置重用，已经分割了的就直接跳过，进入下一个分割
            reuseExistingChunk: true,
            //值越高权重越高
            priority: 0,
            test: function(module, moduleChunks){
              console.log(module)
              return true
            },
            name: "last",
            chunks: "all"
        }
    }
  }
}
```

> `runtimeChunk`即为`webpack`的运行时环境，比如`webpack`的模块定义，初始化等。

```json
//值的格式如下
//如果是object，只能是name:(entry)=>{return '自定义名称'}
//如果是string，single|multiple
//single(只生成一个runtime文件，多个entry的话按output设置的格式命名)
//multiple(为每个entry生成一个runtime文件，并按下面的方式命名)
//默认是false，在每个文件中包含
runtimeChunk: object|string|boolean
//如果未自定义返回名称，而且设置的是string，则生成的名称为runtime+automaticNameDelimiter+entryName使用
```

> `runtimeChunk`大概长这样：


```js
/******/ (function(modules) { // webpackBootstrap
/******/  // install a JSONP callback for chunk loading
/******/  function webpackJsonpCallback(data) {
/******/    var chunkIds = data[0];
/******/    var moreModules = data[1];
/******/    var executeModules = data[2];
/******/
/******/    // add "moreModules" to the modules object,
/******/    // then flag all "chunkIds" as loaded and fire callback
/******/    var moduleId, chunkId, i = 0, resolves = [];
/******/    for(;i < chunkIds.length; i++) {
/******/      chunkId = chunkIds[i];
/******/      if(installedChunks[chunkId]) {
/******/        resolves.push(installedChunks[chunkId][0]);
/******/      }
/******/      installedChunks[chunkId] = 0;
/******/    }
/******/    for(moduleId in moreModules) {
/******/      if(Object.prototype.hasOwnProperty.call(moreModules, moduleId)) {
/******/        modules[moduleId] = moreModules[moduleId];
/******/      }
/******/    }
/******/    if(parentJsonpFunction) parentJsonpFunction(data);
/******/
/******/    while(resolves.length) {
/******/      resolves.shift()();
/******/    }
/******/
/******/    // add entry modules from loaded chunk to deferred list
/******/    deferredModules.push.apply(deferredModules, executeModules || []);
/******/
/******/    // run deferred modules when all chunks ready
/******/    return checkDeferredModules();
/******/  };
/******/  function checkDeferredModules() {
/******/    var result;
/******/    for(var i = 0; i < deferredModules.length; i++) {
/******/      var deferredModule = deferredModules[i];
/******/      var fulfilled = true;
/******/      for(var j = 1; j < deferredModule.length; j++) {
/******/        var depId = deferredModule[j];
/******/        if(installedChunks[depId] !== 0) fulfilled = false;
/******/      }
/******/      if(fulfilled) {
/******/        deferredModules.splice(i--, 1);
/******/        result = __webpack_require__(__webpack_require__.s = deferredModule[0]);
/******/      }
/******/    }
/******/    return result;
/******/  }
/******/
/******/  // The module cache
/******/  var installedModules = {};
/******/
/******/  // object to store loaded and loading chunks
/******/  // undefined = chunk not loaded, null = chunk preloaded/prefetched
/******/  // Promise = chunk loading, 0 = chunk loaded
/******/  var installedChunks = {
/******/    "runtime~main": 0
/******/  };
/******/
/******/  var deferredModules = [];
/******/
/******/  // The require function
/******/  function __webpack_require__(moduleId) {
/******/
/******/    // Check if module is in cache
/******/    if(installedModules[moduleId]) {
/******/      return installedModules[moduleId].exports;
/******/    }
/******/    // Create a new module (and put it into the cache)
/******/    var module = installedModules[moduleId] = {
/******/      i: moduleId,
/******/      l: false,
/******/      exports: {}
/******/    };
/******/
/******/    // Execute the module function
/******/    modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
/******/
/******/    // Flag the module as loaded
/******/    module.l = true;
/******/
/******/    // Return the exports of the module
/******/    return module.exports;
/******/  }
/******/
/******/
/******/  // expose the modules object (__webpack_modules__)
/******/  __webpack_require__.m = modules;
/******/
/******/  // expose the module cache
/******/  __webpack_require__.c = installedModules;
/******/
/******/  // define getter function for harmony exports
/******/  __webpack_require__.d = function(exports, name, getter) {
/******/    if(!__webpack_require__.o(exports, name)) {
/******/      Object.defineProperty(exports, name, { enumerable: true, get: getter });
/******/    }
/******/  };
/******/
/******/  // define __esModule on exports
/******/  __webpack_require__.r = function(exports) {
/******/    if(typeof Symbol !== 'undefined' && Symbol.toStringTag) {
/******/      Object.defineProperty(exports, Symbol.toStringTag, { value: 'Module' });
/******/    }
/******/    Object.defineProperty(exports, '__esModule', { value: true });
/******/  };
/******/
/******/  // create a fake namespace object
/******/  // mode & 1: value is a module id, require it
/******/  // mode & 2: merge all properties of value into the ns
/******/  // mode & 4: return value when already ns object
/******/  // mode & 8|1: behave like require
/******/  __webpack_require__.t = function(value, mode) {
/******/    if(mode & 1) value = __webpack_require__(value);
/******/    if(mode & 8) return value;
/******/    if((mode & 4) && typeof value === 'object' && value && value.__esModule) return value;
/******/    var ns = Object.create(null);
/******/    __webpack_require__.r(ns);
/******/    Object.defineProperty(ns, 'default', { enumerable: true, value: value });
/******/    if(mode & 2 && typeof value != 'string') for(var key in value) __webpack_require__.d(ns, key, function(key) { return value[key]; }.bind(null, key));
/******/    return ns;
/******/  };
/******/
/******/  // getDefaultExport function for compatibility with non-harmony modules
/******/  __webpack_require__.n = function(module) {
/******/    var getter = module && module.__esModule ?
/******/      function getDefault() { return module['default']; } :
/******/      function getModuleExports() { return module; };
/******/    __webpack_require__.d(getter, 'a', getter);
/******/    return getter;
/******/  };
/******/
/******/  // Object.prototype.hasOwnProperty.call
/******/  __webpack_require__.o = function(object, property) { return Object.prototype.hasOwnProperty.call(object, property); };
/******/
/******/  // __webpack_public_path__
/******/  __webpack_require__.p = "";
/******/
/******/  var jsonpArray = window["webpackJsonp"] = window["webpackJsonp"] || [];
/******/  var oldJsonpFunction = jsonpArray.push.bind(jsonpArray);
/******/  jsonpArray.push = webpackJsonpCallback;
/******/  jsonpArray = jsonpArray.slice();
/******/  for(var i = 0; i < jsonpArray.length; i++) webpackJsonpCallback(jsonpArray[i]);
/******/  var parentJsonpFunction = oldJsonpFunction;
/******/
/******/
/******/  // run deferred modules from other chunks
/******/  checkDeferredModules();
/******/ })
/************************************************************************/
/******/ ([]);
```