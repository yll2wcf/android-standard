#DeviceUtils

```java
public class DeviceUtils {

    /**
     * 获取设备唯一标识
     *
     * @param context
     * @return
     */
    public static String getDeviceId(final Context context) {
        final String[] szImei = new String[1];
        RxPermissions.getInstance(context)
                .request(Manifest.permission.READ_PHONE_STATE)
                .subscribe(new Action1<Boolean>() {
                    @Override
                    public void call(Boolean aBoolean) {
                        if (aBoolean) {
                            TelephonyManager TelephonyMgr = (TelephonyManager) context.getSystemService(Context.TELEPHONY_SERVICE);
                            szImei[0] = TelephonyMgr.getDeviceId();
                        }
                    }
                });

        WifiManager wm = (WifiManager) context.getApplicationContext().getSystemService(Context.WIFI_SERVICE);
        String m_szWLANMAC = wm.getConnectionInfo().getMacAddress();


        String m_szLongID = szImei[0] + m_szWLANMAC;

        return MDUtils.md5(m_szLongID);
    }

}

```

需要权限
```xml

    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
    <uses-permission android:name="android.permission.READ_PHONE_STATE" />
```