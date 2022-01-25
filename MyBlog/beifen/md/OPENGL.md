# OPENGL

继承GLSurfaceView

//init过程中

初始化，设置EGL版本2.0

setEGLContextClinetVersion(2);

//Todo 设置渲染器EGL开始一个GLThread,start run {renderer ,onSurfaceCreate,onSurfaceChanged,onDrawFrame}

setRenderer(new MyGLrenderer(this));

其中MyGLrenderer实现GLSurfaceView.Renderer三个接口

设置渲染模式

RENDERMODE_WHEN_DIRTY按需渲染，有帧数据的时候才会去渲染，效率较高，后面需要手动调一次

RENDERMODE_CONTINUOUSLY每隔16ms读取更新一次如果没有显示显示上一帧，自动刷新

setRenderMode(RENDERMODE_WHEN_DIRTY);



自定义渲染器

实现GLSurfaceView.Renderer三个函数

实现SurfaceTexture.OnFrameAvailableListener唯一的一个实现函数

onFrameAvailable，用于手动渲染，有可用的数据回调 

手动渲染指令mGLSurfaceView.requestRender();



//创建是回调此函数

1.onSurfaceCreate(GL10 g1, EGLConfig cinfig)

//获取纹理id

int[] mTextureID = new int[1];

//1.长度2.纹理id，是一个数组3.offset，0就是下标为0

GLES20.glGenTextures(mTextureID.length, mTextureID, 0);

//绑定纹理id，实例化纹理对象

SurfaceTexture mSurfaceTexture  = new SurfaceTexture(mTextureID[0]); 

//绑定好此监听SurfaceTexture.OnFrameAvailableListener

mSurfaceTexture .setOnFrameAvailableListener(this);

mScreenFilter = new ScreenFilter(myGLSurfaceView.getContext());



//改变时回调此函数

2.onSurfaceChanged((GL10 g1, int width, int height)

mcameraHelper.startPreview(mSurfaceTexture);

mScreenFilter.onReady(width, height);

![image-20210804224230347](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20210804224230347.png)



//绘制每一帧图像时回调此函数

3.onDrawFrame(GL10 gl)

//每次清空之前的，类似擦黑板

glclearColor(255,0,0,0);//屏幕清理成红色

//GL_COLOR_BUFFER_BIT颜色缓冲区

//GL_DEPTH_BUFFER_BIT深度缓冲区

//GL_STENCIL_BUFFER_BIT模型缓冲区

glclear(GL_COLOR_BUFFER_BIT);//需要传一个标记

//绘制摄像头数据

mSurfaceTexture.updateTexImage();//将纹理图像更新为图像流中最新的帧数据

//画布，矩阵数据

float[] mtx = new float[16];

mSurfaceTexture.getTransforMatrix(mtx);//native赋值

mScreenFilter.onDrawFrame(mTextureID[0], mtx);



//opengl相机方式设置

mcamera.setPreviewDisplay(mSurfaceHolder);

必须设置成mcamera.setPreviewTexture(surfaceTexture);





//专门显示的类，滤镜类ScreenFilter

ScreenFilter(Context context)

//readTextFileFormResource将glsl转化成字符串

String vertexSource = readTextFileFormResource(context, R.raw.vertex);

String fragmentSource = readFileFromSource(context, R.raw.ragment);

//todo配置顶点着色器

//1.1创建顶点着色器

int vShaderId = glCreateShader(GL_VERTEX_SHADER);

//1.2绑定着色器源代码到着色器（加载着色器的代码）

glShaderSource(vShaderId, vertexSource );

//1.3编译，用于内部检查glsl的代码

glCompileShader(vShaderId);

int []status = new int[1];

glGrtShaderiv(vShaderId, GL_COMPILE_STATUS, status, 0);

if (status[0] != GL_TRUE)

{throw new IllegalStateException("顶点着色器配置失败！");}



//todo配置片元着色器

//2.1创建片元着色器

int fShaderId = glCreateShader(GL_FRAGMENT_SHADER);

//2.2绑定着色器源代码到着色器（加载着色器的代码）

glShaderSource(fShaderId , fragmentSource );

//2.3编译，用于内部检查glsl的代码

glCompileShader(fShaderId);

glGrtShaderiv(fShaderId, GL_COMPILE_STATUS, status, 0);

if (status[0] != GL_TRUE)

{throw new IllegalStateException("片元着色器配置失败！");}

//配置着色器程序

//3.1创建一个着色器程序

mProgram = glCreateProgram();

//3.2将前面配置的顶点和片元着色器附加到新的程序上

glAttachShader(mProgram ,vShaderId);

glAttachShader(mProgram ,fShaderId);

//3.3链接着色器

glLinkProgram(mProgram );//mProgram 是我们的成果

glGetShaderiv(mProgram , GL_LINK_STATUS, status, 0);

if (status[0] != GL_TRUE)

{throw new IllegalStateException("着色器程序链接失败！");}

//4释放，删除着色器，只需要mProgram 

glDeleteShader(vShaderId);

glDeleteShader(fShaderId);



//获取变量的索引值

vPostion = glGetAttribLocation(mProgram ,"vPostion");//顶点着色器的索引值

vCoord = glGetAttribLocation(mProgram ,"vCoord");//顶点着色器纹理坐标索引值

vMatrix = glGetUniformLocation(mProgram ,"vMatrix");//顶点着色器变量矩阵的索引值

//片元着色器里面的如下

vTexture = glGetUniformLocation(mProgram ,"vTexture");//片元着色器采样器

//NIO buffer缓存

//顶点坐标系缓存（顶点：位置+排版）

//4个坐标，xy分量，float占字节数

mVertexBuffer = ByteBuffer.allocationDirect(4 * 2 * 4)

.order(ByteOrder.nativeOrder())//使用本地字节序，不考虑大端小端

.asFloatBuffer();

mVertexBuffer.clear();//清除一下

float[] v = {//opengl世界坐标

-1.0f, -1.0f,

1.0f, -1.0f,

-1.0f, 1.0f,

1.0f, 1.0f,

}

mVertexBuffer.put(v);

//纹理坐标系缓存（顶点：上色+成果）

//4个坐标，xy分量，float占字节数

mTextureBuffer = ByteBuffer.allocationDirect(4 * 2 * 4)

.order(ByteOrder.nativeOrder())//使用本地字节序，不考虑大端小端

.asFloatBuffer();

mTextureBuffer.clear();//清除一下

/*float[] t = {//android屏幕坐标

-0.0f, 1.0f,

1.0f, 1.0f,

0.0f, 0.0f,

1.0f, 0.0f,

}*/

//旋转180

float[] t = {//android屏幕坐标

1.0f, 0.0f

0.0f, 0.0f,

1.0f, 1.0f,

0.0f, 1.0f,

}

mTextureBuffer.put(t);



onReady(int width,int height)

{mWidth= width;mHeight=height}



//绘制操作

onDrawFrame(int mTextureID, float[] mtx)

//设置视窗大小，从0开始

glViewport(0, 0, mWidth, mHeight);

glUseProgram(mProgram);//执行着色器程序

//todo顶点坐标赋值

//顶点坐标赋值NIObuffer用它需要归零

mVertexBuffer.position(0);

//传值，把float[] 值传递给顶点着色器，把mVertexBuffer传递到vPosition，size每次两个xy，stride=0代表坐标值不跳步

glVertexAttribPoint(vPosition, 2， GL_FLOAT, false, 0, mVertexBuffer);//用于将mVertexBuffer的内容映射到vPosition中

//激活

glEnableVertexAttribArray(vPosition);

//todo 纹理坐标赋值同上

mTextureBuffer.position(0);

//传值，把float[] 值传递给顶点着色器，把mVertexBuffer传递到vPosition，size每次两个xy，stride=0代表坐标值不跳步

glVertexAttribPoint(vCoord, 2， GL_FLOAT, false, 0, mTextureBuffer);//用于将mVertexBuffer的内容映射到vPosition中

//激活

glEnableVertexAttribArray(vCoord);

//todo变换矩阵把mtx矩阵数据传递给vMatrix

glUniformMatrix4fv(vMatrix, 1, false, mtx, 0);

//片元着色器

glActiveTexture(GL_TEXTURE0);//激活图层

glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, mTextureID);

//传递数据给片元着色器的采样器

glUniformli(vTexture, 0);

glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);//通知opengl绘制，从0开始共四个点绘制



着色器语言glsl

顶点着色器vertex.glsl

attribute vec4 vPostion;//顶点坐标，相机的四个点位置

attribute vec4 vCoord；//纹理坐标，用来图形上色

uniform mat4 vMatrix;//变换矩阵，4*4

varying vec2 aCoord;//易变变量，从顶点坐标系传递到片源着色器的数据变量,把这个最终的计算成果给片元着色器

void main()

{

//内置变量，vec4类型，gl_Position表示顶点着色器定点位置

//gl_FragColor，vec4类型，表示片元着色器中颜色

gl_Position = vPostion；//确定位置

//为了兼容所有设备，碎片网格细节

aCoord = (vMatrix * vCoord).xy;

}

片元着色器fragment.glsl

#extension GL_OES_EGL_image_external:require

varying vec2 aCoord;//拿到最终计算成果，给片元着色器才能上色

//float 数据的精度 （precision lowp低精度 precision mediump中精度 precision highp高精度）

precision mediump float；

//根据上面的数据精度，写下面的采样器相机数据

//uniform sampler2D vTexture;由于我们用的是安卓相机，不能使用

uniform samplerExternalOES vTexture;//安卓采样器，需要导一个包

void main（）

{

//内置函数texture2D（采样器(涉及到精度)，坐标）采样指定位置的纹理

gl_FragColor = texture2D(vTexture, aCoord);



//添加美颜下面是灰度图

vec4 rgba = texture2D(vTexture, aCoord);

float gray = (0.30 *rgba.a +0.59*rgba.g+0.11*rgba.b);

gl_fragColor = vec4(gray, gray, gray, 1.0);

}

![image-20210807142542243](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20210807142542243.png)
