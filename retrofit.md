#Retrofit 基础


实际开发过程中一般都会选择一些网络框架提升开发效率。随着Google对HttpClient 摒弃和Volley框架的逐渐没落，OkHttp开始异军突起，而Retrofit则对OkHttp进行了强制依赖，可以简单理解Retroifit在OKHttp基础上进一步完善。


>Retrofit是由[Square](https://github.com/square)公司出品的针对于Android和Java的类型安全的Http客户端，目前推出了2.0+的版本。

Retrofit框架项目地址：[https://github.com/square/retrofit](https://github.com/square/retrofit)。
Retrofit官方文档地址: [http://square.github.io/retrofit/](http://square.github.io/retrofit/)

##使用Retrofit

 接下来我们来学习下如何使用Retrofit。
首先需要在app/build.gradle添加依赖。

```py
dependencies {
    //...
    //retrofit
    compile 'com.squareup.retrofit2:retrofit:2.1.0'
    //如果用到gson解析 需要添加下面的依赖
    compile 'com.squareup.retrofit2:converter-gson:2.1.0'
}
```
我们以查号码归属地接口为例 https://www.juhe.cn/docs/api/id/11

Retrofit不能直接使用，需要进行初始化，在这里创建NetWork.java

```java
public class NetWork {
    private static Retrofit retrofit;
  
    /**返回Retrofit*/
    public static Retrofit getRetrofit(){
        if(retrofit==null){
            Retrofit.Builder builder = new Retrofit.Builder();//创建Retrfit构建器
            retrofit = builder.baseUrl("http://apis.juhe.cn/") //指定网络请求的baseUrl
                    .addConverterFactory(GsonConverterFactory.create())//返回的数据通过Gson解析
                    .build();
        }
        return retrofit;
    }
}
```

>Retrofit需要之地baseUrl，往往一个项目中有很多接口，接口都使用相同的服务器地址，这时候可以把接口地址相同的部分抽取到baseUrl中，Retrofit扩展性极好，可以指定返回的数据通过Gson解析，前提你需要保证项目中有Gson框架和com.squareup.retrofit2:converter-gson:2.1.0的依赖。


除了通过Gson解析还可以使用其它的方式解析，需要的依赖也不同，有如下几种：

* Gson: com.squareup.retrofit:converter-gson
* Jackson: com.squareup.retrofit:converter-jackson
* Moshi: com.squareup.retrofit:converter-moshi
* Protobuf: com.squareup.retrofit:converter-protobuf
* Wire: com.squareup.retrofit:converter-wire
* Simple XML: com.squareup.retrofit:converter-simplexml 

>Retrofit需要把Http的请求接口封装到一个接口文件中。

```java
public interface NetInterface {
    //获取号码归属地，返回来类型是Bean, 需要两个参数分别为phone何key
    @GET("mobile/get")
    Call<Bean> getAddress(@Query("phone") String phone, @Query("key") String key);
}
```
其中Bean是根据请求的结果创建的对象.

方法前添加@GET注解表示当前请求是Get方式请求，链接的地址是baseUrl+"mobile/get"，baseUrl在初始化Retrofit的时候指定了，拼到一起就是 [http://apis.juhe.cn/mobile/get](http://apis.juhe.cn/mobile/get)。
对于 Retrofit 2.0中新的URL定义方式，这里是我的建议：

* baseUrl: 总是以 /结尾
* url: 不要以 / 开头 

因为如果不是这种方式，拼装后的结果和你期望的是不一样的，详情参考官方文档。

>除了Get请求还有下面几种请求方式

* @POST    表明这是post请求    
* @PUT    表明这是put请求    
* @DELETE    表明这是delete请求    
* @PATCH    表明这是一个patch请求，该请求是对put请求的补充，用于更新局部资源    
* @HEAD    表明这是一个head请求    
* @OPTIONS    表明这是一个option请求 
* @HTTP    通用注解,可以替换以上所有的注解，其拥有三个属性：method，path，hasBody    

最后的HTTP通用注解写法比较特殊，请求可以代替之前的请求。下面的写法和之前的@GET效果是一样的。

```java
/**
 * method 表示请的方法，不区分大小写
 * path表示路径
 * hasBody表示是否有请求体
 */
@HTTP(method = "get",path = "mobile/get",hasBody = false)
Call<Bean> getAddress(@Query("phone") String phone, @Query("key") String key);
```

@Quert表示查询参数，用于GET查询，注解里的字符串是参数的key值，参数会自动拼装到Url后面。
除了上面的注解，再给大家介绍几种不同的注解。

##常用的注解

@Url：使用全路径复写baseUrl，适用于非统一baseUrl的场景。示例代码：
```java
@GET Call<ResponseBody> XXX(@Url String url);
```
@Streaming:用于下载大文件。示例代码：
```java
@Streaming @GET Call<ResponseBody> downloadFileWithDynamicUrlAsync(@Url String fileUrl);
  
  
//获取数据的代码
ResponseBody body = response.body();
long fileSize = body.contentLength();
InputStream inputStream = body.byteStream();
```
@Path：URL占位符，用于替换和动态更新,相应的参数必须使用相同的字符串被@Path进行注释
```java
//实际请求地址会给句groupId的值发生变化--> http://baseurl/group/groupId/users
@GET("group/{id}/users") Call<List<User>> groupList(@Path("id") int groupId);
```

@QueryMap:查询参数，和@Query类似，区别就是后面需要Map集合参数。示例代码：
```java
Call<List<News>> getNews((@QueryMap(encoded=true) Map<String, String> options);
```

@Body:用于POST请求体，将实例对象根据转换方式转换为对应的json字符串参数，这个转化方式是GsonConverterFactory定义的。 示例代码：
```java
@POST("add")
Call<List<User>> addUser(@Body User user);
```
@Field，@FieldMap:Post方式传递简单的键值对，需要添加@FormUrlEncoded表示表单提交 

```java
@FormUrlEncoded @POST("user/edit") Call<User> updateUser(@Field("first_name") String first, @Field("last_name") String last);
```

@Part，@PartMap：用于POST文件上传，其中@Part MultipartBody.Part代表文件，@Part(“key”) RequestBody代表参数，需要添加@Multipart表示支持文件上传的表单。
```java
@Multipart
   @POST("upload")
   Call<ResponseBody> upload(@Part("description") RequestBody description,
                             @Part MultipartBody.Part file);
```

## 完成请求实例

了解了Retrofit，我们用Retrofit请求完成请求，Retrofit使用起来比较省事，核心代码如下所示：

```java
//初始化Retrofit,加载接口
NetInterface netInterface = NetWork.getRetrofit().create(NetInterface.class);
//请求接口
netInterface.getAddress(editText.getText().toString(),"你的app key")
        .enqueue(new Callback<Bean>() {
            @Override
            public void onResponse(Call<Bean> call, Response<Bean> response) {
                //请求成功
                Bean bean = response.body();
                //...
            }
   
            @Override
            public void onFailure(Call<Bean> call, Throwable t) {
                //请求失败
            }
        });

```


Retrofit会自动在子线程中进行网络请求，请求结束切换到主线程中，而且内部使用了线程池，对网络请求的缓存控制的也非常到位，网络响应速度也是很快的，使用起来非常的爽!

##RxJava和Retrofit结合

>RxJava非常强大，就连Retrofit都要抱下他的大腿，Retrofit也可以用RxJava方式进行网络请求，只需要对上面的代码进行改造即可。

首先添加框架依赖。

```py
dependencies {
  //...
       
    compile 'io.reactivex:rxandroid:1.2.1'
    compile 'io.reactivex:rxjava:1.1.6'
   
    compile 'com.google.code.gson:gson:2.8.0'
   
    compile 'com.squareup.retrofit2:retrofit:2.1.0'
    //如果用到gson解析 需要添加下面的依赖
    compile 'com.squareup.retrofit2:converter-gson:2.1.0'
    //Retrofit使用RxJava需要的依赖
    compile 'com.squareup.retrofit2:adapter-rxjava:2.1.0'
   
}

```

修改Retrofit初始化的代码:

```java
public class NetWork {
    private static Retrofit retrofit;
  
    /**返回Retrofit*/
    public static Retrofit getRetrofit(){
        if(retrofit==null){
            //创建Retrfit构建器
            Retrofit.Builder builder = new Retrofit.Builder();
            //指定网络请求的baseUrl
            retrofit = builder.baseUrl("http://apis.juhe.cn/")
                    //返回的数据通过Gson解析
                    .addConverterFactory(GsonConverterFactory.create())
                    //使用RxJava模式
                    .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                    .build();
        }
        return retrofit;
    }
}
```

上面代码我们通过，添加代码```addCallAdapterFactory(RxJavaCallAdapterFactory.create()) ```就变成了使用RxJava模式。

接口也需要修改,把方法的返回值类型由Call改成了RxJava中的Observable。

```java
public interface NetInterface {
    //获取号码归属地，返回来类型是Bean, 需要两个参数分别为phone何key
    @GET("mobile/get")
    Observable<Bean> getAddress(@Query("phone") String phone, @Query("key") String key);
}
```

接下来修改最终网络请求的代码，可以改成RxJava方式了。

```java
  NetInterface netInterface = NetWork.getRetrofit().create(NetInterface.class);
    //RxJava方式
    netInterface.getAddress(editText.getText().toString(),"你的app key")
            .subscribeOn(Schedulers.io())//设置网络请求在子线程中
            .observeOn(AndroidSchedulers.mainThread())// 回调在主线程中
            .subscribe(new Action1<Bean>() {
                @Override
                public void call(Bean bean) {
                    //请求成功
                }
            }, new Action1<Throwable>() {
                @Override
                public void call(Throwable throwable) {
                    //请求失败
                }
            });
```
