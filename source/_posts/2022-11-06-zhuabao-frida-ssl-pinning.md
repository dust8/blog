---
title: 抓包之frida解决ssl pinning
date: 2022-11-06 06:15:31
tags:
  - 抓包
  - frida
---
想看下视频网站的 `只看TA` 怎么实现的, 所以想抓包. 情况如下:
- 优酷电脑没有, 手机有不收费
- 腾讯电脑没有, 手机没有, 好像收费才看到
- 爱奇艺电脑有

开始没看爱奇艺, 所以才抓的优酷包, 而优酷包有证书绑定, 抓不到包.
爱奇艺的格式是
```json
{
  "id": 0,
  "stars": ["200072305"],
  "points": [
    {
      "startTime": 192,
      "endTime": 233
    },
    {
      "startTime": 321,
      "endTime": 334
    },
    {
      "startTime": 811,
      "endTime": 1033
    },
    {
      "startTime": 1248,
      "endTime": 1405
    },
    {
      "startTime": 1500,
      "endTime": 1581
    },
    {
      "startTime": 1621,
      "endTime": 1741
    }
  ],
  "totalTime": 634
}
```
`200072305` 是演员的id. `points` 就是她在的视频段. 上面例子就是代表有6个片段包含该演员. `startTime`, `endTime` 就是片段的开始时间和结束时间. `totalTime`就是片段总的时长, 单位是秒, 所以在页面显示时长为`10:34`.

## 手机端
去github下载`frida-server`包, 需要下载手机相应的架构, 在手机端运行.
例如电脑上的安卓模拟器下载`frida-server-16.0.2-android-x86_64.xz`, android代表运行在安卓手机, x86_64代表模拟器是电脑端的64位. 解压后把文件推送到手机上.
```
# 推送文件
adb push frida-server-16.0.2-android-x86_64 /data/local/tmp
```

```
# 运行frida-server
adb shell 
cd /data/local/tmp
# 修改权限, 手机需要root
su
chmod 777 frida-server-16.0.2-android-x86_64
# 运行
./frida-server-16.0.2-android-x86_64
```

## 电脑端
安装frida
```
pip install frida-tools
```

启动要查看的程序, 然后运行, 可以找到程序的包名
```
adb shell dumpsys window w |findstr \/ |findstr name=
```

hook文件
```js
# hookssl.js
Java.perform(function() {
 
/*
hook list:
1.SSLcontext
2.okhttp
3.webview
4.XUtils
5.httpclientandroidlib
6.JSSE
7.network\_security\_config (android 7.0+)
8.Apache Http client (support partly)
9.OpenSSLSocketImpl
10.TrustKit
11.Cronet
*/
 
	// Attempts to bypass SSL pinning implementations in a number of
	// ways. These include implementing a new TrustManager that will
	// accept any SSL certificate, overriding OkHTTP v3 check()
	// method etc.
	var X509TrustManager = Java.use('javax.net.ssl.X509TrustManager');
	var HostnameVerifier = Java.use('javax.net.ssl.HostnameVerifier');
	var SSLContext = Java.use('javax.net.ssl.SSLContext');
	var quiet_output = false;
 
	// Helper method to honor the quiet flag.
 
	function quiet_send(data) {
 
		if (quiet_output) {
 
			return;
		}
 
		send(data)
	}
 
 
	// Implement a new TrustManager
	// ref: https://gist.github.com/oleavr/3ca67a173ff7d207c6b8c3b0ca65a9d8
	// Java.registerClass() is only supported on ART for now(201803). 所以android 4.4以下不兼容,4.4要切换成ART使用.
	/*
06-07 16:15:38.541 27021-27073/mi.sslpinningdemo W/System.err: java.lang.IllegalArgumentException: Required method checkServerTrusted(X509Certificate[], String, String, String) missing
06-07 16:15:38.542 27021-27073/mi.sslpinningdemo W/System.err:     at android.net.http.X509TrustManagerExtensions.<init>(X509TrustManagerExtensions.java:73)
        at mi.ssl.MiPinningTrustManger.<init>(MiPinningTrustManger.java:61)
06-07 16:15:38.543 27021-27073/mi.sslpinningdemo W/System.err:     at mi.sslpinningdemo.OkHttpUtil.getSecPinningClient(OkHttpUtil.java:112)
        at mi.sslpinningdemo.OkHttpUtil.get(OkHttpUtil.java:62)
        at mi.sslpinningdemo.MainActivity$1$1.run(MainActivity.java:36)
*/
	var X509Certificate = Java.use("java.security.cert.X509Certificate");
	var TrustManager;
	try {
		TrustManager = Java.registerClass({
			name: 'org.wooyun.TrustManager',
			implements: [X509TrustManager],
			methods: {
				checkClientTrusted: function(chain, authType) {},
				checkServerTrusted: function(chain, authType) {},
				getAcceptedIssuers: function() {
					// var certs = [X509Certificate.$new()];
					// return certs;
					return [];
				}
			}
		});
	} catch (e) {
		quiet_send("registerClass from X509TrustManager >>>>>>>> " + e.message);
	}
 
 
 
 
 
	// Prepare the TrustManagers array to pass to SSLContext.init()
	var TrustManagers = [TrustManager.$new()];
 
	try {
		// Prepare a Empty SSLFactory
		var TLS_SSLContext = SSLContext.getInstance("TLS");
		TLS_SSLContext.init(null, TrustManagers, null);
		var EmptySSLFactory = TLS_SSLContext.getSocketFactory();
	} catch (e) {
		quiet_send(e.message);
	}
 
	send('Custom, Empty TrustManager ready');
 
	// Get a handle on the init() on the SSLContext class
	var SSLContext_init = SSLContext.init.overload(
		'[Ljavax.net.ssl.KeyManager;', '[Ljavax.net.ssl.TrustManager;', 'java.security.SecureRandom');
 
	// Override the init method, specifying our new TrustManager
	SSLContext_init.implementation = function(keyManager, trustManager, secureRandom) {
 
		quiet_send('Overriding SSLContext.init() with the custom TrustManager');
 
		SSLContext_init.call(this, null, TrustManagers, null);
	};
 
	/*** okhttp3.x unpinning ***/
 
 
	// Wrap the logic in a try/catch as not all applications will have
	// okhttp as part of the app.
	try {
 
		var CertificatePinner = Java.use('okhttp3.CertificatePinner');
 
		quiet_send('OkHTTP 3.x Found');
 
		CertificatePinner.check.overload('java.lang.String', 'java.util.List').implementation = function() {
 
			quiet_send('OkHTTP 3.x check() called. Not throwing an exception.');
		}
 
	} catch (err) {
 
		// If we dont have a ClassNotFoundException exception, raise the
		// problem encountered.
		if (err.message.indexOf('ClassNotFoundException') === 0) {
 
			throw new Error(err);
		}
	}
 
	// Appcelerator Titanium PinningTrustManager
 
	// Wrap the logic in a try/catch as not all applications will have
	// appcelerator as part of the app.
	try {
 
		var PinningTrustManager = Java.use('appcelerator.https.PinningTrustManager');
 
		send('Appcelerator Titanium Found');
 
		PinningTrustManager.checkServerTrusted.implementation = function() {
 
			quiet_send('Appcelerator checkServerTrusted() called. Not throwing an exception.');
		}
 
	} catch (err) {
 
		// If we dont have a ClassNotFoundException exception, raise the
		// problem encountered.
		if (err.message.indexOf('ClassNotFoundException') === 0) {
 
			throw new Error(err);
		}
	}
 
	/*** okhttp unpinning ***/
 
 
	try {
		var OkHttpClient = Java.use("com.squareup.okhttp.OkHttpClient");
		OkHttpClient.setCertificatePinner.implementation = function(certificatePinner) {
			// do nothing
			quiet_send("OkHttpClient.setCertificatePinner Called!");
			return this;
		};
 
		// Invalidate the certificate pinnet checks (if "setCertificatePinner" was called before the previous invalidation)
		var CertificatePinner = Java.use("com.squareup.okhttp.CertificatePinner");
		CertificatePinner.check.overload('java.lang.String', '[Ljava.security.cert.Certificate;').implementation = function(p0, p1) {
			// do nothing
			quiet_send("okhttp Called! [Certificate]");
			return;
		};
		CertificatePinner.check.overload('java.lang.String', 'java.util.List').implementation = function(p0, p1) {
			// do nothing
			quiet_send("okhttp Called! [List]");
			return;
		};
	} catch (e) {
		quiet_send("com.squareup.okhttp not found");
	}
 
	/*** WebView Hooks ***/
 
	/* frameworks/base/core/java/android/webkit/WebViewClient.java */
	/* public void onReceivedSslError(Webview, SslErrorHandler, SslError) */
	var WebViewClient = Java.use("android.webkit.WebViewClient");
 
	WebViewClient.onReceivedSslError.implementation = function(webView, sslErrorHandler, sslError) {
		quiet_send("WebViewClient onReceivedSslError invoke");
		//执行proceed方法
		sslErrorHandler.proceed();
		return;
	};
 
	WebViewClient.onReceivedError.overload('android.webkit.WebView', 'int', 'java.lang.String', 'java.lang.String').implementation = function(a, b, c, d) {
		quiet_send("WebViewClient onReceivedError invoked");
		return;
	};
 
	WebViewClient.onReceivedError.overload('android.webkit.WebView', 'android.webkit.WebResourceRequest', 'android.webkit.WebResourceError').implementation = function() {
		quiet_send("WebViewClient onReceivedError invoked");
		return;
	};
 
	/*** JSSE Hooks ***/
 
	/* libcore/luni/src/main/java/javax/net/ssl/TrustManagerFactory.java */
	/* public final TrustManager[] getTrustManager() */
	/* TrustManagerFactory.getTrustManagers maybe cause X509TrustManagerExtensions error  */
	// var TrustManagerFactory = Java.use("javax.net.ssl.TrustManagerFactory");
	// TrustManagerFactory.getTrustManagers.implementation = function(){
	//     quiet_send("TrustManagerFactory getTrustManagers invoked");
	//     return TrustManagers;
	// }
 
	var HttpsURLConnection = Java.use("javax.net.ssl.HttpsURLConnection");
	/* libcore/luni/src/main/java/javax/net/ssl/HttpsURLConnection.java */
	/* public void setDefaultHostnameVerifier(HostnameVerifier) */
	HttpsURLConnection.setDefaultHostnameVerifier.implementation = function(hostnameVerifier) {
		quiet_send("HttpsURLConnection.setDefaultHostnameVerifier invoked");
		return null;
	};
	/* libcore/luni/src/main/java/javax/net/ssl/HttpsURLConnection.java */
	/* public void setSSLSocketFactory(SSLSocketFactory) */
	HttpsURLConnection.setSSLSocketFactory.implementation = function(SSLSocketFactory) {
		quiet_send("HttpsURLConnection.setSSLSocketFactory invoked");
		return null;
	};
	/* libcore/luni/src/main/java/javax/net/ssl/HttpsURLConnection.java */
	/* public void setHostnameVerifier(HostnameVerifier) */
	HttpsURLConnection.setHostnameVerifier.implementation = function(hostnameVerifier) {
		quiet_send("HttpsURLConnection.setHostnameVerifier invoked");
		return null;
	};
 
	/*** Xutils3.x hooks ***/
	//Implement a new HostnameVerifier
	var TrustHostnameVerifier;
	try {
		TrustHostnameVerifier = Java.registerClass({
			name: 'org.wooyun.TrustHostnameVerifier',
			implements: [HostnameVerifier],
			method: {
				verify: function(hostname, session) {
					return true;
				}
			}
		});
 
	} catch (e) {
		//java.lang.ClassNotFoundException: Didn't find class "org.wooyun.TrustHostnameVerifier"
		quiet_send("registerClass from hostnameVerifier >>>>>>>> " + e.message);
	}
 
	try {
		var RequestParams = Java.use('org.xutils.http.RequestParams');
		RequestParams.setSslSocketFactory.implementation = function(sslSocketFactory) {
			sslSocketFactory = EmptySSLFactory;
			return null;
		}
 
		RequestParams.setHostnameVerifier.implementation = function(hostnameVerifier) {
			hostnameVerifier = TrustHostnameVerifier.$new();
			return null;
		}
 
	} catch (e) {
		quiet_send("Xutils hooks not Found");
	}
 
	/*** httpclientandroidlib Hooks ***/
	try {
		var AbstractVerifier = Java.use("ch.boye.httpclientandroidlib.conn.ssl.AbstractVerifier");
		AbstractVerifier.verify.overload('java.lang.String', '[Ljava.lang.String', '[Ljava.lang.String', 'boolean').implementation = function() {
			quiet_send("httpclientandroidlib Hooks");
			return null;
		}
	} catch (e) {
		quiet_send("httpclientandroidlib Hooks not found");
	}
 
	/***
android 7.0+ network_security_config TrustManagerImpl hook
apache httpclient partly
***/
	var TrustManagerImpl = Java.use("com.android.org.conscrypt.TrustManagerImpl");
	// try {
	//     var Arrays = Java.use("java.util.Arrays");
	//     //apache http client pinning maybe baypass
	//     //https://github.com/google/conscrypt/blob/c88f9f55a523f128f0e4dace76a34724bfa1e88c/platform/src/main/java/org/conscrypt/TrustManagerImpl.java#471
	//     TrustManagerImpl.checkTrusted.implementation = function (chain, authType, session, parameters, authType) {
	//         quiet_send("TrustManagerImpl checkTrusted called");
	//         //Generics currently result in java.lang.Object
	//         return Arrays.asList(chain);
	//     }
	//
	// } catch (e) {
	//     quiet_send("TrustManagerImpl checkTrusted nout found");
	// }
 
	try {
		// Android 7+ TrustManagerImpl
		TrustManagerImpl.verifyChain.implementation = function(untrustedChain, trustAnchorChain, host, clientAuth, ocspData, tlsSctData) {
			quiet_send("TrustManagerImpl verifyChain called");
			// Skip all the logic and just return the chain again :P
			//https://www.nccgroup.trust/uk/about-us/newsroom-and-events/blogs/2017/november/bypassing-androids-network-security-configuration/
			// https://github.com/google/conscrypt/blob/c88f9f55a523f128f0e4dace76a34724bfa1e88c/platform/src/main/java/org/conscrypt/TrustManagerImpl.java#L650
			return untrustedChain;
		}
	} catch (e) {
		quiet_send("TrustManagerImpl verifyChain nout found below 7.0");
	}
	// OpenSSLSocketImpl
	try {
		var OpenSSLSocketImpl = Java.use('com.android.org.conscrypt.OpenSSLSocketImpl');
		OpenSSLSocketImpl.verifyCertificateChain.implementation = function(certRefs, authMethod) {
			quiet_send('OpenSSLSocketImpl.verifyCertificateChain');
		}
 
		quiet_send('OpenSSLSocketImpl pinning')
	} catch (err) {
		quiet_send('OpenSSLSocketImpl pinner not found');
	}
	// Trustkit
	try {
		var Activity = Java.use("com.datatheorem.android.trustkit.pinning.OkHostnameVerifier");
		Activity.verify.overload('java.lang.String', 'javax.net.ssl.SSLSession').implementation = function(str) {
			quiet_send('Trustkit.verify1: ' + str);
			return true;
		};
		Activity.verify.overload('java.lang.String', 'java.security.cert.X509Certificate').implementation = function(str) {
			quiet_send('Trustkit.verify2: ' + str);
			return true;
		};
 
		quiet_send('Trustkit pinning')
	} catch (err) {
		quiet_send('Trustkit pinner not found')
	}
 
	try {
		//cronet pinner hook
		//weibo don't invoke
 
		var netBuilder = Java.use("org.chromium.net.CronetEngine$Builder");
 
		//https://developer.android.com/guide/topics/connectivity/cronet/reference/org/chromium/net/CronetEngine.Builder.html#enablePublicKeyPinningBypassForLocalTrustAnchors(boolean)
		netBuilder.enablePublicKeyPinningBypassForLocalTrustAnchors.implementation = function(arg) {
 
			//weibo not invoke
			console.log("Enables or disables public key pinning bypass for local trust anchors = " + arg);
 
			//true to enable the bypass, false to disable.
			var ret = netBuilder.enablePublicKeyPinningBypassForLocalTrustAnchors.call(this, true);
			return ret;
		};
 
		netBuilder.addPublicKeyPins.implementation = function(hostName, pinsSha256, includeSubdomains, expirationDate) {
			console.log("cronet addPublicKeyPins hostName = " + hostName);
 
			//var ret = netBuilder.addPublicKeyPins.call(this,hostName, pinsSha256,includeSubdomains, expirationDate);
			//this 是调用 addPublicKeyPins 前的对象吗? Yes,CronetEngine.Builder
			return this;
		};
 
	} catch (err) {
		console.log('[-] Cronet pinner not found')
	}
});

```
运行com.youku.phone 是x酷的包名, 改成你抓包的
```
frida -U -l hookssl.js -f com.youku.phone
```

在抓包软件已经可以看到之前抓不到的 https 包了.



## 参考链接
- [frida/frida](https://github.com/frida/frida)
- [r0ysue/r0capture](https://github.com/r0ysue/r0capture)
- [Fiddler真机模拟器抓包+frida突破ssl pinning](https://blog.csdn.net/qq_18893835/article/details/121461497)
- [安卓应用层协议/框架通杀抓包：实战篇](https://www.anquanke.com/post/id/228709)