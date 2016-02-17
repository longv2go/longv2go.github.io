---
layout: post
title:  React Native Android 通信原理
---

React Native (Android)内置了一个用于解析JavaScript(以下简称JS)脚本的框架，方便把Java类暴漏给JS调用，具体的使用方法[参见](https://facebook.github.io/react-native/docs/native-modules-android.html#content)，这篇文章就用来研究一下Java和JS的通信原理，JS是如何调用Java的。

#总体结构
当初始化阶段，Java端会把所有要暴漏的Java类的信息封装成Config传给JS，然后根据Config生成对应Java类的Javascript镜像对象，以及要暴漏的方法，在JS中调用这个镜像对象的方法就会被转发到对应的Java对象上，如下所示
![](https://raw.githubusercontent.com/longv2go/longv2go.github.io/master/postImages/react-and-arc.png)

JS的代码总要被解析执行，那么React是在哪里执行JS的呢？React并没有通过webview去执行JS代码，它是通过Jni调用c++代码通过Javascriptcore来执行JS的，首先来看看生成so依赖的的文件，代码在react-native/ReactAndroid/src/main/jni目录下。*（用NDK编译在Android上运行的c/c++代码，关于NDK请自行google）*
![reactnativejni](https://raw.githubusercontent.com/longv2go/longv2go.github.io/master/postImages/react-and-lib.png)
其中OnLoad.cpp很关键，里面通过Jni映射了本地的方法到Java中，是Java和C++之间的桥梁。在Java中主要通过ReactBridge.java来调用C++，NativeModulesReactCallback类是C++调用Java的桥梁。

例如以下代码，截取自OnLoad.cpp的JNI_OnLoad方法*(这个方法会在Java载入so文件的时候由Jni首先调用)*

```C++
registerNatives("com/facebook/react/bridge/JSCJavaScriptExecutor", {
    makeNativeMethod("initialize", executors::createJSCExecutor),
});
```
意思是把Java中的JSCJavaScriptExecutor类的initialize方法映射为executors::createJSCExecutor的C++方法，这样当在Java中调用initialize就会在C++中执行executors::createJSCExecutor。
#Java端初始化
在第一个Activity创建的时候开始进行整个Brdige的Java端的初始化，流程图如下
![](https://raw.githubusercontent.com/longv2go/longv2go.github.io/master/postImages/react-and-java-init.png)

####初始化主要做几件事情
1. 创建JSCJavaScriptExecutor,这个是个C++包装类，会调用到C++的executors::createJSCExecutor()
2. 创建NativeModuleRegistry管理所有的要暴漏给JS的Java类，暴漏给JS的java类的搜集是通过ReactActivity中的getPackages实现的，详看上图
3. 创建ReactBridge对象，这个对象也是个C++桥梁对象，用来调用C++代码，创建过程会调用到bridge::create()方法
4. 创建config(包含了要暴漏的所有java类的信息，json格式)，并通过bridge设置到JS环境中的__fbBatchedBridgeConfig变量，这样在JS端就可以通过这个变量来获取所有的Java类信息了，然后根据config生产对应的镜像对象。

	config格式如下：
	
	```json
	{
	    "remoteModuleConfig": {
	        "MyToastAndroid": {
	            "moduleID": 14,
	            "methods": {
	                "show": {
	                    "methodID": 0,
	                    "type": "remote"
	                }
	            },
	            "constants": {
	                "LONG": 1,
	                "SHORT": 0
	            }
	        },...
	    }
	}
	```


Java端还会创建一个CatalystInstanceImpl对象，这个对象用来管理所有的NativeModules以及与C++通信的桥梁ReactBrdige,类图结构如下:
![React 类图](https://raw.githubusercontent.com/longv2go/longv2go.github.io/master/postImages/react-and-class.jpg)
####几个重要的类
1. NativeModuleRegistry, 维护一个mModuleInstances数组，这个数组的顺序很重要，因为这和在JS端维护的镜像对象的数组是一致的当JS调用Java的时候实际上传递的正是在这个数组中的索引
2. NativeModuleReactCallBack， C++回调Java的对象，这个对象会在创建ReactBridge的时候传递给C++，当JS调用Java的方法的时候会调用这个类的方法
3. ReactBridge，调用C++的桥梁

最后catalystInstance.runJSBundle()开启JS端的初始化流程

#JS端的初始化
和React Native iOS的JS初始化是一样的，因为Android和iOS的react是同享一份JS代码的，在react命令生成的react native工程的node_modules目录下面存放着所有JS的模块。在编译的时候会把所有的JS模块合并成一个大的JS文件。初始化就是在JS环境中执行这个文件。其中MessageQueue.js, BatchedBridge.js和NativeModules.js三个文件是关于JS bridge的。初始化流程如下图![react-native 通信原理](https://raw.githubusercontent.com/longv2go/longv2go.github.io/master/postImages/react-and-js.jpg)

在遍历RemoteModules的时候需要为每一个映射对象生成Java暴漏的方法，因为JS是不支持消息转发，如果调用了没有实现的方法，那么就直接生成一个错误，所以要知道每一个暴漏的Module要暴漏的方法，在JS端预先生成对应的实现。在Java端初始化的时候已经在JS中注入了config信息，包括了要暴漏的类和方法名，足已生成镜像对象了。MessageQueue.js中的_genMethod方法中为每一个映射对象生成相应的方法实现。最后生成方法如下：

```JavaScript
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

当调用一个镜像对象的方法，就会调用到_nativeCall方法，而参数就是闭包生成的时候捕获的module和method等, 在Java端和JS端会保存一份关于暴漏的Java类对象信息的数组，这俩分数组的顺序是相同的，而 _nativeCall中的参数就是要调用的Java类在数组中的索引，这样在Java端就可以通过索引找到要调用的Java类了。在JS端这个数组是MessageQueue的modulesConfig,Java端是NativeModuleRegistry的mModuleInstances。


#JS调用Java流程
JS会在调用native方法的时候调用 ```_nativeCall```然后调用```global.nativeFlushQueueImmediate(this._queue);```,其中nativeFlushQueueImmediate方法会调用到C++中，是JS调用C++的桥梁

nativeFlushQueueImmediate方法是在C++中的JSCExecutor.cpp中注册的，我们先来看看JSCExecutor的创建过程，如下图
![](https://raw.githubusercontent.com/longv2go/longv2go.github.io/master/postImages/react-and-callback.png)

在JSCExecutor的构造方法中调用了```installGlobalFunction(m_context, "nativeFlushQueueImmediate", nativeFlushQueueImmediate);```，这样就在JS环境中注册了nativeFlushQueueImmediate方法，当在JS中调用了nativeFlushQueueImmediate就会执行JSCExecutor的nativeFlushQueueImmediate C++方法，然后调用 ```executor->flushQueueImmediate(resStr);```,如上图所示，会回调到 OnLoad.cpp中的dispatchCallbacksToJava()方法，上图中红框中是采用了C++的闭包写法，[参考](http://blog.csdn.net/anzhsoft/article/details/17414665)


```
	dispatchCallbacksToJava ---> makeJavaCall() ---> env->CallVoidMethod()
```
最后调用到CallVoidMethod的jni方法，这样就从C++调用到了Java代码了，传入的CallVoidMethod的callback参数就是在创建ReactBrdige的时候传入的NativeModuleReactCallback的java对象对应的jni对象,而gCallbackMethod就是call方法，这样就调用到了Java类NativeModuleReactCallback的call方法。哇哦~终于回到java了~,Java在通过反射最后调用实际的java方法。

#总结
本文只是列出了整个Bridge比较难于理解的部分以及流程，想要详细了解具体原理还需要自己看代码，如果遇到代码中不明白的地方可以参考本文。关于React Native iOS的Objective-C和JS的通信原理请[参考](http://127.0.0.1:4000/2016/01/20/react-native%E9%80%9A%E4%BF%A1%E5%8E%9F%E7%90%86.html). *转载请标明出处*