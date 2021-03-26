---
layout: post
title: 修改APK来允许用户CA
---

Android 从某个版本起（7？）就默认 App 不认用户添加的 Certificate，导致无法抓 HTTPS 包。可以通过修改 App 的 APK 来绕过。

主题内容来自[这个文章](https://hurricanelabs.com/blog/modifying-android-apps-to-allow-tls-intercept-with-user-cas/)。这里讲一下实践中遇到几个坑。

提取APK，用 `apktool` 解包：
```
apktool d app.apk
```
修改 `AndroidManifest.xml` ：

```xml
<?xml version="1.0" encoding="utf-8" standalone="no"?><manifest ...>
	...
    <application ... android:networkSecurityConfig="@xml/network_security_config">
		...
    </application>
</manifest>
```
然后增加个 `res/xml/network_security_config.xml` 文件，内容如下：
```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="true">
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

重新用 `apktool` 打包：
```
apktool b --use-aapt2 app -o app-modified.apk
```
`--use-aapt2`理论上可选，但是我有的APK会失败，所以加上吧。

用 OpenSSL 生成一个 key 和 cert ，这里不赘述直接全贴上来。可以复用：
```
openssl genrsa -out key 1024
openssl pkcs8 -topk8 -in key -out key.pkcs8 -outform DER -nocrypt
openssl req -x509 -key key -out cert.pem -days 3650 -nodes -subj "/CN=example.com"
```

下一步是用 `zipalign` 来对齐。这个工具和下面那个都是 Android Studio 带的。网上能找到别人打包的，但是我这里老遇到问题，还是装个 AS 吧。

理论上不一定需要`-p`，但是我有的APK不加就装不上。[参考](https://stackoverflow.com/a/66295045/3939155)
```
zipalign -p 4 app-modified.apk aligned.apk
```

最后用 `apksigner` 打包（网上有很多用 `jarsigner` 打包的教程，我这里就用 Google 官方的了）：
```
apksigner sign --key key.pkcs8 --cert cert.pem --out signed.apk aligned.apk
```
