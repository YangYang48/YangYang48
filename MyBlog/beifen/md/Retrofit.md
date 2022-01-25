Retrofit

Retrofit是一个RESTful的http网络请求框架的封装，网络请求本质上是Okhttp完成，而Retrofit仅负责网络请求接口的封装

**动态代理，注解，反射**

```java
public interface WeatherApi{
    @POST("/v3/weather/weatherInfo")
    @FormUrlEncoded
    Call<ResponseBody> postWeather(@Field("city") String city, @Field("key") String key);
    
    @GET("/v3/weather/weatherInfo")
    Call<ResponseBody> getWeather(@Query("city") String city, @Query("key") String key);
}
```



```java
Retrofit retrofit = new Retrofit.Builder().baseUrl("https://restapi.amap.com").build();
WeatherApi weatherApi = retrofit.create(WeatherApi.class);
```

类似okhttp

```java
//同步请求
Response response = okhttpClient.newCall(request).execute();
```

```java
//异步请求
Response response = okhttpClient.newCall(request).enqueue(callback);
```





返回200才是网络请求成功的，其余404,200,300都是不正常的

使用建造者模式：将一个复杂对象的构建和它的表示进行分离，可以使使用者不必知道内部组成的细节，

builder里面有两个属性

里面有两个HttpUrl baseUrl和okhttp3.Call.Factory callFactory

如果不对其http进行客制化，这个接口callFactory为null

![image-20211118222622870](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211118222622870.png)

call接口里面的子接口

唯一的实现就是okhttpClient

![image-20211118222714151](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211118222714151.png)



主要逻辑

![image-20211118223423434](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211118223423434.png)

Annotation：注解默认都实现了Annotation接口



![image-20211118224452593](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20211118224452593.png)

post有请求头，get没有请求体

