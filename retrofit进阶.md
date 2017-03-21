#Retrofit进阶

这篇文章介绍点进阶的用法。

##打印网络日志

在开发阶段，为了方便调试，我们需要查看网络日志。因为`Retrofit2.0+`底层是采用的`OKHttp`请求的。可以给OKHttp设置拦截器，用来打印日志。
首先可以在`app/build.gradle`中添加依赖，这是官方的日志拦截器。
```py
compile 'com.squareup.okhttp3:logging-interceptor:3.3.0'
```
然后在代码中设置：
```java
    public static Retrofit getRetrofit() {
        //如果mRetrofit为空  或者服务器地址改变 重新创建
        if (mRetrofit == null) {
            OkHttpClient httpClient;
            OkHttpClient.Builder builder=new OkHttpClient.Builder();
            
            //阶段分为开发和发布阶段，当前为开发阶段设置拦截器
            if (BuildConfig.DEBUG) {
                HttpLoggingInterceptor logging = new HttpLoggingInterceptor();
                //设置拦截器级别
                logging.setLevel(HttpLoggingInterceptor.Level.BODY);
                builder.addInterceptor(logging);
            }
            httpClient=builder.build();
            //构建Retrofit
            mRetrofit = new Retrofit.Builder()
                    //配置服务器路径
                    .baseUrl(mServerUrl)
                    //返回的数据通过Gson解析
                    .addConverterFactory(GsonConverterFactory.create())
                    //配置回调库，采用RxJava
                    .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                    //设置OKHttp模板
                    .client(httpClient)
                    .build();
        }
        return mRetrofit;
    }
```
当处于开发阶段的时候，设置监听日志的拦截器。拦截有4个级别，分别是

1. BODY
2. HEADERS
3. BASIC
4. NONE

其中`BODY`输出的日志是最全的。


##添加相同的请求参数
为了更好的管理迭代版本，一般每次发起请求的时候都传输当前程序的版本号到服务器。
有些项目我们每次还会传用户id，token令牌等相同的参数。

**如果在每个请求的接口都添加这些参数太繁琐。Retrofit可以通过拦截器添加相同的请求参数，无需再每个接口添加了。**

>步骤一，自己拦截器

```java
public class CommonInterceptor implements Interceptor {
    @Override
    public Response intercept(Interceptor.Chain chain) throws IOException {
        Request oldRequest = chain.request();

        // 添加新的参数
        HttpUrl.Builder authorizedUrlBuilder = oldRequest.url()
                .newBuilder()
                .scheme(oldRequest.url().scheme())
                .host(oldRequest.url().host())
                .addQueryParameter("device_type", "1")
                .addQueryParameter("version", BuildConfig.VERSION_NAME)
                .addQueryParameter("token", PreUtils.getString(R.string.token))
                .addQueryParameter("userid", PreUtils.getString(R.string.user_id));

        // 新的请求
        Request newRequest = oldRequest.newBuilder()
                .method(oldRequest.method(), oldRequest.body())
                .url(authorizedUrlBuilder.build())
                .build();

        return chain.proceed(newRequest);
    }
}

```
实现原理就是拦截之前的请求，添加完参数，再传递新的请求。这个位置我添加了四个公共的参数。
然后再Retrofit初始化的时候配置。
```java
        if (mRetrofit == null) {
            OkHttpClient httpClient;
            OkHttpClient.Builder builder=new OkHttpClient.Builder();
            //添加公共参数
            builder.addInterceptor(new CommonInterceptor());
            httpClient=builder.build();
            //构建Retrofit
            mRetrofit = new Retrofit.Builder()
                    //....
                    .client(httpClient)
                    .build();
        }
```

##处理约定错误

除了常见的404，500等异常，网络请求中我们往往还会约定些异常，比如token失效，账号异常等等。

以token失效为例，每次请求我们都需要验证是否失效，如果在每个接口都处理一遍错误就有点太繁琐了。

我们可以统一处理下错误。

>步骤一，Retrofit初始化时添加自定义转化器

```java
mRetrofit = new Retrofit.Builder()
        //配置服务器路径
      baseUrl(mServerUrl)
      //配置回调库，采用RxJava
     .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
     //配置转化库，默认是Gson，这里修改了。
     .addConverterFactory(ResponseConverterFactory.create())
     .client(httpClient)
     .build();
```

>步骤二 创建ResponseConverterFactory

步骤一 `ResponseConverterFactory`这个类是需要我们自己创建的。
```java
public class ResponseConverterFactory extends Converter.Factory {
    /**
     * Create an instance using a default {@link Gson} instance for conversion. Encoding to JSON and
     * decoding from JSON (when no charset is specified by a header) will use UTF-8.
     */
    public static ResponseConverterFactory create() {
        return create(new Gson());
    }

    /**
     * Create an instance using {@code gson} for conversion. Encoding to JSON and
     * decoding from JSON (when no charset is specified by a header) will use UTF-8.
     */
    public static ResponseConverterFactory create(Gson gson) {
        return new ResponseConverterFactory(gson);
    }

    private final Gson gson;

    private ResponseConverterFactory(Gson gson) {
        if (gson == null) throw new NullPointerException("gson == null");
        this.gson = gson;
    }

    @Override
    public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
                                                            Retrofit retrofit) {
      //  TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
        return new GsonResponseBodyConverter<>(gson, type);
    }

    @Override
    public Converter<?, RequestBody> requestBodyConverter(Type type,
                                                          Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
        TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
        return new GsonRequestBodyConverter<>(gson, adapter);
    }
}

```
这里面我们自定义了请求和响应时解析JSON的转换器——`GsonRequestBodyConverter`和`GsonResponseBodyConverter`。

其中`GsonRequestBodyConverter` 负责处理请求时传递JSON对象的格式，不需要额外处理任何事，直接使用默认的GSON解析。代码我直接贴出来：
```java
final class GsonRequestBodyConverter<T> implements Converter<T, RequestBody> {
    private static final MediaType MEDIA_TYPE = MediaType.parse("application/json; charset=UTF-8");
    private static final Charset UTF_8 = Charset.forName("UTF-8");

    private final Gson gson;
    private final TypeAdapter<T> adapter;

    GsonRequestBodyConverter(Gson gson, TypeAdapter<T> adapter) {
        this.gson = gson;
        this.adapter = adapter;
    }

    @Override public RequestBody convert(T value) throws IOException {
        Buffer buffer = new Buffer();
        Writer writer = new OutputStreamWriter(buffer.outputStream(), UTF_8);
        JsonWriter jsonWriter = gson.newJsonWriter(writer);
        adapter.write(jsonWriter, value);
        jsonWriter.close();
        return RequestBody.create(MEDIA_TYPE, buffer.readByteString());
    }
}
```



`GsonResponseBodyConverter `负责把响应的数据转换成JSON格式，这个我们需要处理一下。

```java

public class GsonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
    private final Gson gson;
    private final Type type;

    GsonResponseBodyConverter(Gson gson, Type type) {
        this.gson = gson;
        this.type = type;
    }

    @Override
    public T convert(ResponseBody value) throws IOException {
        String response = value.string();
        try {
            Log.i("YLlibrary", "response>>> "+response);
            //ResultResponse 只解析result字段
            BaseInfo baseInfo = gson.fromJson(response, BaseInfo.class);
            if (baseInfo.getHeader().getCode().equals("1")) {
                //正确
                return gson.fromJson(response, type);
            } else {
                //ErrResponse 将msg解析为异常消息文本 错误码可以自己指定.
                throw new ResultException(-1024, baseInfo,response);
            }
        } finally {
        }
    }

}

```
这种情况只是应用于后台接口数据统一的情况。比如我们项目的格式是这样的
```json
      {
          header : {"message":"token失效","code":"99"}
          data : {}
      }
```
当code值是1的时候，表示正确，其它数字表示错误。只有正确的时候data才会有内容。

这里我用BaseInfo解析这个JSON：
```java
public class BaseInfo {

    /**
     * header : {"message":"用户名或密码错误","code":"0"}
     * data : {}
     */

    private HeaderBean header;

    public HeaderBean getHeader() {
        return header;
    }

    public void setHeader(HeaderBean header) {
        this.header = header;
    }


    public static class HeaderBean {
        /**
         * message : 用户名或密码错误
         * code : 0
         */

        private String message;
        private String code;

        public String getMessage() {
            return message;
        }

        public void setMessage(String message) {
            this.message = message;
        }

        public String getCode() {
            return code;
        }

        public void setCode(String code) {
            this.code = code;
        }
    }
}
```
服务器返回的数据实体对象全部继承`BaseInfo` 只是data内容不一样。

`ResultException`这个类用于捕获服务器约定的错误类型
```java
/**
 * 这个类用于捕获服务器约定的错误类型
 */
public class ResultException extends RuntimeException {

    private int errCode = 0;
    private BaseInfo info;
    private String response;
    public ResultException(int errCode, BaseInfo info,String response) {
        super(info.getHeader().getMessage());
        this.info=info;
        this.errCode = errCode;
        this.response=response;
    }

    public String getResponse() {
        return response;
    }

    public void setResponse(String response) {
        this.response = response;
    }

    public int getErrCode() {
        return errCode;
    }

    public BaseInfo getBaseInfo(){
        return info;
    }
}
```

>最后定义Retrofit处理异常的代码

```java
public abstract class AbsAPICallback<T> extends Subscriber<T> {

    //对应HTTP的状态码
    private static final int UNAUTHORIZED = 401;
    private static final int FORBIDDEN = 403;
    private static final int NOT_FOUND = 404;
    private static final int REQUEST_TIMEOUT = 408;
    private static final int INTERNAL_SERVER_ERROR = 500;
    private static final int BAD_GATEWAY = 502;
    private static final int SERVICE_UNAVAILABLE = 503;
    private static final int GATEWAY_TIMEOUT = 504;
    //出错提示
    private final String networkMsg;
    private final String parseMsg;
    private final String unknownMsg;

    protected AbsAPICallback(String networkMsg, String parseMsg, String unknownMsg) {
        this.networkMsg = networkMsg;
        this.parseMsg = parseMsg;
        this.unknownMsg = unknownMsg;
    }
    public  AbsAPICallback(){
        networkMsg="net error(联网失败)";
        parseMsg="json parser error(JSON解析失败)";
        unknownMsg="unknown error(未知错误)";
    }
    ProgressBar progressBar;
    public  AbsAPICallback(ProgressBar progressBar){
        this();
        this.progressBar=progressBar;
    }
    @Override
    public void onError(Throwable e) {
        Throwable throwable = e;
        //获取最根源的异常
        while(throwable.getCause() != null){
            e = throwable;
            throwable = throwable.getCause();
        }

        ApiException ex;
        if (e instanceof HttpException){             //HTTP错误
            HttpException httpException = (HttpException) e;
            ex = new ApiException(e, httpException.code());
            switch(httpException.code()){
                case UNAUTHORIZED:
                case FORBIDDEN:
                  //  onPermissionError(ex);          //权限错误，需要实现
                   // break;
                case NOT_FOUND:
                case REQUEST_TIMEOUT:
                case GATEWAY_TIMEOUT:
                case INTERNAL_SERVER_ERROR:
                case BAD_GATEWAY:
                case SERVICE_UNAVAILABLE:
                default:
                    ex.setDisplayMessage(networkMsg);  //均视为网络错误
                    onNetError(ex);
                    break;
            }
        } else if (e instanceof ResultException){    //服务器返回的错误
            ResultException resultException = (ResultException) e;
            onResultError(resultException);
        } else if (e instanceof JsonParseException
                || e instanceof JSONException
                || e instanceof ParseException){
            ex = new ApiException(e, ApiException.PARSE_ERROR);
            ex.setDisplayMessage(parseMsg);            //均视为解析错误
            onNetError(ex);
        } else {
            ex = new ApiException(e, ApiException.UNKNOWN);
            ex.setDisplayMessage(unknownMsg);          //未知错误
            onNetError(ex);
        }
    }
    static long time;
    protected  void onNetError(ApiException e){
        long currentTime=System.currentTimeMillis();
        if(currentTime-time>3000){  //防止连续反馈
            time=currentTime;
            UIUtils.showToast("网络加载失败");
        }
        e.printStackTrace();
        onApiError(e);
    }
    /**
     * 错误回调
     */
    protected  void onApiError(ApiException ex){
        Log.i("YLLibrary","onApiError");
        if(progressBar!=null)
            UIUtils.runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    progressBar.setVisibility(View.GONE);
                    progressBar=null;
                }
            });
    }

//    /**
//     * 权限错误，需要实现重新登录操作
//     */
//    protected void  onPermissionError(ApiException ex){
//        ex.printStackTrace();
//    }

    /**
     * 服务器返回的错误
     */
    protected  synchronized  void onResultError(ResultException ex){
//        if(ex.getErrCode()== XApplication.API_ERROR){
//            UIUtils.getContext().onApiError(); //可以用来处理Token失效
//            return ;
//        }
        if(ConstantValue.TOKEN_ERROR.equals(ex.getBaseInfo().getHeader().getCode())
                &&!TextUtils.isEmpty(PreUtils.getString(R.string.token))){ //验证token是否为空是为了防止连续两次请求
            PreUtils.putString(R.string.user_id,null);
            PreUtils.putString(R.string.token,null);
            PreUtils.putString(R.string.orgDistrict,null);
            if(BaseActivity.runActivity!=null){
                Intent intent = new Intent(UIUtils.getContext(), LoginActivity.class);
                if(BaseActivity.runActivity instanceof MainActivity){
                    MainActivity activity= (MainActivity) BaseActivity.runActivity;
                    int tabIndex=activity.getCurrentTab();
                    //activity.switchCurrentTab(0);
                    activity.startActivityForResult(intent,tabIndex+10);
                }else {
                    BaseActivity.runActivity.startActivity(intent);
                }
            }
        }


        Log.i("YLLibrary","resultError");
        if(ex.getBaseInfo()!=null&&!TextUtils.isEmpty(ex.getBaseInfo().getHeader().getMessage()))
            UIUtils.showToast(ex.getBaseInfo().getHeader().getMessage());

        ApiException apiException = new ApiException(ex, ex.getErrCode());
        onApiError(apiException);
    }

    @Override
    public void onCompleted() {
        Log.i("YLLibrary","onCompleted");
        if(progressBar!=null)
            UIUtils.runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    progressBar.setVisibility(View.GONE);
                    progressBar=null;
                }
            });
    }

}
```

实际接口请求的代码，使用自定义异常回调的类——`AbsAPICallback`就可以统一处理异常:
```java
        ApiRequestManager.createApi().problemDetail(dataBean.getId())
                .compose(ApiRequestManager.<QuestionDetailInfo>applySchedulers())
                .subscribe(new AbsAPICallback<QuestionDetailInfo>() {
                    @Override
                    public void onNext(QuestionDetailInfo baseInfo) {
                        fillData(baseInfo);
                    }
                });
```