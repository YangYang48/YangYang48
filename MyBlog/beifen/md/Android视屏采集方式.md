Android视屏采集方式

1.设置摄像头参数，包括：camera一些属性摄像头聚焦之类（可选），设置预览模式，一般有几种，NV21和NV12，设置摄像头宽和高，设置缓存和数据缓冲区，设置画面，启动预览。

2.通过在初始化和回调设置addCallbackBuffer，会出现byte类型的回调，而对于这个回调数据即为可以做NV21--->I420的处理，最后处理完的数据，通过编码，传入rtmp的流中

3.有了data回调数据，通过mCamera.setPreviewDisplay(mSurfaceHolder)中的画面绑定传到控件SurfaceView来显示。



具体代码如下

1.设置摄像头参数

```java
/**
 * 开始预览
 * 我们相机采集的的画面的数据 yuv420类型有很多  例如： nv21， i420， nv12， nvxx， ....
 *
 * 简单说下：下节课画图说
 *
 * w * h * 3 / 2
 *
 YUV 420 子集的之一
 * 4 * 4
 * y y y y
 * y y y y
 * y y y y
 * y y y y
 * u u u u
 * v v v v
 *
 * YUV 420 子集的之一
 * 4 * 4
 * y y y y
 * y y y y
 * y y y y
 * y y y y
 * u v u v
 * u v u v
 *
 * 我们为什么一定用YUV，不用RGBA？
 * 答：1.在各个算法领域上 YUV 更高效，算法更成熟
 *    2.RGBA8888 4个字节32位， YUV RGAB 好像是一半
 *    3.黑白电视 Y只有明亮度 黑白色，   彩色电视 UV 色度和饱和度
 */
private void startPreview() {
    try {
        // 获得camera对象
        mCamera = Camera.open(mCameraId);
        // 配置camera的属性
        Camera.Parameters parameters = mCamera.getParameters();
        // 设置预览数据格式为nv21
        parameters.setPreviewFormat(ImageFormat.NV21); // yuv420类型的子集
        // 这是摄像头宽、高
        setPreviewSize(parameters);
        // 设置摄像头 图像传感器的角度、方向
        setPreviewOrientation(parameters);
        mCamera.setParameters(parameters);
        buffer = new byte[mWidth * mHeight * 3 / 2]; // 请看什么的细节
        // 数据缓存区
        mCamera.addCallbackBuffer(buffer);
        mCamera.setPreviewCallbackWithBuffer(this);
        // 设置预览画面
        mCamera.setPreviewDisplay(mSurfaceHolder); // SurfaceView 和 Camera绑定
        if (mOnChangedSizeListener != null) { // 你的宽和高发生改变，就会回调此接口
            mOnChangedSizeListener.onChanged(mWidth, mHeight);
        }
        // 开启预览
        mCamera.startPreview();
    } catch (Exception ex) {
        ex.printStackTrace();
    }
}
```

2.画布回调（可选）

通过mSurfaceHolder.addCallback(this);绑定，回调SurfaceHolder.Callback，最终显示三个回调，画布创建、画布改变（改变画布大小或者摄像头方向），画布销毁

```java
@Override
public void surfaceCreated(SurfaceHolder holder) { }

@Override
public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
    // 释放摄像头
    stopPreview();
    // 开启摄像头
    startPreview();
}

@Override
public void surfaceDestroyed(SurfaceHolder holder) {
    stopPreview(); // 只要画面不可见，就必须释放，因为预览耗电 耗资源
}
```



3.预览回调

通过对camera.addCallbackBuffer(buffer)的绑定buffer数据，实现Camera.PreviewCallback的回调，（必须）

```java
@Override
public void onPreviewFrame(byte[] data, Camera camera) {
    // TODO 作业：data没有做旋转处理
    // 这个只是画面的旋转，但是数据不会旋转，你还需要额外处理
    if (mPreviewCallback != null) {
        mPreviewCallback.onPreviewFrame(data, camera); // byte[] data == nv21 ===> C++层 ---> 流媒体服务器
    }
    camera.addCallbackBuffer(buffer);
}
```





注意：在调用Camera.startPreview()接口前，我们需要setPreviewCallbackWithBuffer，而setPreviewCallbackWithBuffer之前我们需要重新addCallbackBuffer，因为setPreviewCallbackWithBuffer 使用时需要指定一个字节数组作为缓冲区，用于预览图像数据 即addCallbackBuffer，然后你在onPerviewFrame中的data才会有值；

3，从上面看来，我们设置addCallbackBuffer的地方有两个，一个是在startPreview之前，一个是在onPreviewFrame中，这两个都需要调用，如果在onPreviewFrame中不调用，那么，就无法继续回调到onPreviewFrame中了。