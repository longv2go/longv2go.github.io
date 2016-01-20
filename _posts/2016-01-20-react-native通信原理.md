---
layout: post
title: "react-native 通信原理"
---


React-native 是Facebook开源的一个用JavaScript开发原生应用的一个框架。它里面包括了一个内置的javascript bridge，可以方便的暴漏oc的方法给js调用，我们就来研究一下它是如何实现的。在本片文章中就简称它为Bridge吧。


#Bridge使用方法
[参考](https://facebook.github.io/react-native/docs/native-modules-ios.html#content)


在JS端会为每一个在OC端暴漏给JS的module生成一个对应的对象，我们称之为映射对象，对这个对象调用的方法会转发给OC端，那么我们要解决以下几个问题

1. 是如何把OC的模块暴漏给JS
2. JS中的映射对象是怎么生成的，里面的方法的实现又是什么
3. 在JS中调用OC的模块是如何进行消息转发的

#初始化
当创建一个RCTBridge对象的时候会进入初始化流程，主要分为俩部分，一个是OC端的初始化，一个JS端的初始化
###OC端Bridge初始化
我们先来看一下```RCT_EXPORT_MODULE```宏定义

```
#define RCT_EXPORT_MODULE(js_name) \
RCT_EXTERN void RCTRegisterModule(Class); \
+ (NSString *)moduleName { return @#js_name; } \
+ (void)load { RCTRegisterModule(self); }

```
这个宏定义是把要暴漏的class在load方法中注册到一个全局的数组中.

初始化流程如下图：
![](https://raw.githubusercontent.com/longv2go/longv2go.github.io/master/postImages/react_native_oc.png)
####loadSource
在xcode编译的时候执行一个脚本把所有的JS文件都合并成一个大的文件bundle.js，在上图中的-loadSource就是去读入这个统一的文件。

####initModules
对所有通过RCT_EXPORT_MODULE暴漏的OC模块进行处理，生成对应的RCTModuleData，并且生成_moduleDataByName字典用来保持所有的ModuleData。RCTModuleData主要用来表示每一个要暴漏的本地Module，实时生成config传给JS端，JS更具config json来生成对应的对象。那么config具体是什么格式呢，稍后在说。

####setUpExecutor
初始化 _javaScriptExecutor，在JSContext中注入俩个关键的方法，nativeRequireModuleConfig，nativeFlushQueueImmediate，这俩个方法会在JS端调用到

1. nativeRequireModuleConfig

	当在JS端要位每一个对应的映射对象生成暴漏的方法的时候，就会调用这个接口，返回所有要暴漏的方法名字
	
2. nativeFlushQueueImmediate，这个方法在JS调用OC的时候会用到。

####moduleConfig
生成一个包括所有要暴漏的Modules列表，并注入到JSContext中的__fbBatchedBridgeConfig变量中，这样JS中就可以知道有多少Module要暴漏了。config的内容如下
```{"remoteModuleConfig":[["ViewController"],["ExportModule"]]}```

####executeSourceCode
这一步就是要在JSContext中执行在第一部中生成的同一个的JS脚本bundle.js，至此就进入了JS端的初始化

###JS端Bridge初始化

在react命令生成的react native工程的node_modules目录下面存放着所有JS的模块。其中MessageQueue.js, BatchedBridge.js和NativeModules.js三个文件是关于JS bridge的。
流程图如下:
![](https://raw.githubusercontent.com/longv2go/longv2go.github.io/master/postImages/react_native_js.jpg)

在遍历RemoteModules的时候需要为每一个映射对象生成OC暴漏的方法，因为JS是不支持类似OC的消息转发，如果调用了没有实现的方法，那么就直接生成一个错误，所以要知道每一个暴漏的Module要暴漏的方法，在JS端预先生成对应的实现。在OC端初始化的时候注入过一个方法nativeRequireModuleConfig，就是用来获取所有的暴漏方法名字的，返回值如下格式：

```
[
  "ExportModule",
  [
    "hello",
    "call"
  ]
]
```
在MessageQueue.js中的_genMethod方法中为每一个映射对象生成相应的方法实现。最后生成方法如下：

```
> NativeModules.ExportModule.hello
< function () {
          for (var _len2 = arguments.length, args = Array(_len2), _key2 = 0; _key2 < _len2; _key2++) {
            args[_key2] = arguments[_key2];
          }

          var lastArg = args.length > 0 ? args[args.length - 1] : null;
          var secondLastArg = args.length > 1 ? args[args.length - 2] : null;
          var hasSuccCB = typeof lastArg === 'function';
          var hasErrorCB = typeof secondLastArg === 'function';
          hasErrorCB && invariant(hasSuccCB, 'Cannot have a non-function arg after a function arg.');
          var numCBs = hasSuccCB + hasErrorCB;
          var onSucc = hasSuccCB ? lastArg : null;
          var onFail = hasErrorCB ? secondLastArg : null;
          args = args.slice(0, args.length - numCBs);
          return self.__nativeCall(module, method, args, onFail, onSucc);
        }
```

#调用过程
基于以上的分析我们知道当调用一个native方法的时候会首先走到__nativeCall，
把所有的调用参数都放入到_queue数组中，然后调用nativeFlushQueueImmediate通知OC端有新的调用，OC就会去取得所有的_queue中的数据进行处理最终调用到本地的方法。 具体流程图如下：

![](https://github.com/longv2go/longv2go.github.io/raw/master/postImages/react_native_call.jpg)

####回调
在JS调用的OC方法中带有回调的时候，JS端把这个回调放在了_callbacks数组中，然后把对应的index作为callbackID传给OC，这样当oc执行完方法，执行回调的时候再把这个callbackID传递回来，JS端在根据callbackID找到回调的fucntion，再执行。




#最后
react native的bridge模块不能运行在webview上，我把bridge模块抽离出来一个独立库，可以运行在webview上，详见[RJSBridge](https://github.com/longv2go/RJSBridge)