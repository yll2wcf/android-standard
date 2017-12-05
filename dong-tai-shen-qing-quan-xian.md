# 动态申请权限

**Android6.0以上权限机制改了，下面的列举的权限都需要动态申请**

![](/assets/动态申请权限.png)

可以使用一些框架完成。

```py
compile 'com.tbruyelle.rxpermissions:rxpermissions:0.7.0@aar'
compile 'io.reactivex:rxjava:1.1.6'
```

下面就是获取手机状态的示例代码。

https://github.com/tbruyelle/RxPermissions



```java
RxPermissions.getInstance(context)
    .request(Manifest.permission.READ_PHONE_STATE)//申请权限
    .subscribe(new Action1<Boolean>() {
        @Override
        public void call(Boolean aBoolean) {
            if (aBoolean) {
                    TelephonyManager TelephonyMgr = (TelephonyManager) context.getSystemService(Context.TELEPHONY_SERVICE);
                    szImei[0] = TelephonyMgr.getDeviceId();
                }
            }
        });
```



