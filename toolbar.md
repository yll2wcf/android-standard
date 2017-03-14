#toolbar

toolbar 里面的文字居中

创建布局 `toolbar.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.v7.widget.Toolbar
    android:id="@+id/toolbar"
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="?attr/actionBarSize"
    android:gravity="center"
    android:background="@drawable/title_bg"
    >

    <TextView
        android:id="@+id/title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerInParent="true"
        android:layout_gravity="center"
        android:gravity="center"
        android:text=""
        android:textColor="@color/white"
        android:textSize="19dp"/>
</android.support.v7.widget.Toolbar>
```

然后再需要用到`toolbar`的布局中使用
```xml
 <include layout="@layout/toolbar"/>
```
