---
layout: post
title:  React Native Android 通信原理
---



React Native (Android)内置了一个用于解析JS脚本的框架，方便把Java类暴漏给JS调用，具体的使用方法[参见](https://facebook.github.io/react-native/docs/native-modules-android.html#content)，这篇文章就用来研究一下Java和JS的通信原理，JS是如何调用Java的。

#总体框架
当初始化阶段，Java端会把所有要暴漏的Native Modules的信息封装成Config传给JS，在JS段会更具Config生成对应Java类的镜像对象，以及暴漏的方法，在JS中调用这个镜像对象的方法就会被转发到对应的Java对象上，如下所示
![](http://127.0.0.1:4000/postImages/react-and-arc.png)

JS的代码总要被解析执行，那么React是在哪里执行JS的呢？React并没有通过webview去执行JS代码，具体原因不清楚，它是通过Jni调用c++代码通过Javascriptcore来执行JS的，首先来看看生成so的文件结构。
![reactnativejni](http://127.0.0.1:4000/postImages/react-and-lib.png)
其中OnLoad.cpp很关键，里面通过Jni映射了本地的方法到Java中，是Java和C++之间的桥梁。在Java中主要通过ReactBridge.java来调用C++，NativeModulesReactCallback类是C++调用Java的桥梁。

例如以下代码，截取自OnLoad.cpp

```C++
registerNatives("com/facebook/react/bridge/JSCJavaScriptExecutor", {
    makeNativeMethod("initialize", executors::createJSCExecutor),
});
```
意思是所把Java中的JSCJavaScriptExecutor类的initialize方法映射为executors::createJSCExecutor的C++方法，这样当在Java中调用initialize就会在C++中执行executors::createJSCExecutor。
C++的代码在react-native/ReactAndroid/src/main/jni目录下。
#Java端初始化
流程图如下
![](http://127.0.0.1:4000/postImages/react-and-java-init.png)

####初始化主要做几件事情
1. 创建JSCJavaScriptExecutor,这个是个C++包装类，会调用到C++的executors::createJSCExecutor()
2. 创建NativeModuleRegistry管理所有的要暴漏给JS的Java类，暴漏给JS的java类的搜集是通过ReactActivity中的getPackages实现的，详看流程图
3. 创建ReactBridge对象，这个对象也是个C++桥梁对象，用来调用C++代码，创建过程会调用到bridge::create()方法
4. 创建config(包含了要暴漏的所有java类的信息，json格式)，并通过bridge设置到JS环境中的__fbBatchedBridgeConfig变量，这样在JS端就可以通过这个变量来获取所有的Java类信息了

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
![React 类图](http://127.0.0.1:4000/postImages/react-and-class.jpg)
####几个重要的类
1. NativeModuleRegistry, 维护一个mModuleInstances数组，这个数组的顺序很重要，因为这和在JS端维护的镜像对象的数组是一致的当JS调用Java的时候实际上传递的正是在这个数组中的索引
2. NativeModuleReactCallBack， C++回调Java的对象，这个对象会在创建ReactBridge的时候传递给C++，当JS调用Java的方法的时候会调用这个类的方法
3. ReactBridge，调用C++的桥梁

最后catalystInstance.runJSBundle()开启JS端的初始化流程

#JS端的初始化
和React Native iOS的JS初始化是一样的，因为Android和iOS的react是同享一份JS代码的，参见[react-native 通信原理](https://longv2go.github.io/2016/01/20/react-native%E9%80%9A%E4%BF%A1%E5%8E%9F%E7%90%86.html)

#JS调用Java流程
JS会在调用native方法的时候调用 ```global.nativeFlushQueueImmediate(this._queue);```（MessageQueue.js）这个方法，其中nativeFlushQueueImmediate方法会调用到C++中，是JS调用C++的桥梁

在JSCExecutor.cpp中 ```installGlobalFunction(m_context, "nativeFlushQueueImmediate", nativeFlushQueueImmediate);```这段代码把本地的nativeFlushQueueImmediate c++方法映射到JS环境的nativeFlushQueueImmediate方法, nativeFlushQueueImmediate会调用```executor->flushQueueImmediate(resStr);``` 然后调用```m_flushImmediateCallback(queueJSON);```,其中m_flushImmediateCallback是在创建JSCExecutor的时候传递过来的，


首先来看C++ Bridge对象的创建过程

```C++
//OnLoad.cpp
static void create(JNIEnv* env, jobject obj, jobject executor, jobject callback,
                   jobject callbackQueueThread) {
  auto weakCallback = createNew<WeakReference>(callback);
  auto weakCallbackQueueThread = createNew<WeakReference>(callbackQueueThread);
  auto bridgeCallback = [weakCallback, weakCallbackQueueThread] (std::vector<MethodCall> calls, bool isEndOfBatch) {
    dispatchCallbacksToJava(weakCallback, weakCallbackQueueThread, std::move(calls), isEndOfBatch);
  };
  auto nativeExecutorFactory = extractRefPtr<JSExecutorFactory>(env, executor);
  auto bridge = createNew<Bridge>(nativeExecutorFactory, bridgeCallback);
  setCountableForJava(env, obj, std::move(bridge));
}
```

在创建Bridge对象的时候传递进去一个闭包对象bridgeCallback，这个回调到dispatchCallbacksToJava，把bridge对象和Java对象obj建立起了关联，在之后的代码中就可以通过```auto bridge = extractRefPtr<Bridge>(env, obj);```来获取刚创建的Bridge对象


