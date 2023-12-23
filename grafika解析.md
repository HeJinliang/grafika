

### CameraCaptureActivity

> 功能：
> 打开Camera1 + OpenGL渲染 + MediaCodec硬编码

主要工具类：

- FullFrameRect

- TextureMovieEncoder

- VideoEncoderCore

- EglSurfaceBase

#### 将相机数据给到 OpenGL

1. 打开相机
    ```
    openCamera(1280, 720);  
    ```
2. 在 GLSurfaceView 的 onSurfaceCreated 回调里 通过工具类 FullFrameRect 获取 OpenGL 给的 纹理ID，通过纹理ID 创建一个 SurfaceTexture
    ```
    mTextureId = mFullScreen.createTextureObject();
    mSurfaceTexture = new SurfaceTexture(mTextureId);
    ```

3. 将这个 SurfaceTexture 设置成相机的输出和预览
    ```
    // 设置 Camera 的输出预览 SurfaceTexture，并开始预览
    private void handleSetSurfaceTexture(SurfaceTexture st) {
        st.setOnFrameAvailableListener(this);
        try {
            mCamera.setPreviewTexture(st);
        } catch (IOException ioe) {
            throw new RuntimeException(ioe);
        }
        mCamera.startPreview();
    }
    ```


#### 将openGL数据给到 MediaCodec

关键代码:
```
mInputWindowSurface = new WindowSurface(mEglCore, mVideoEncoder.getInputSurface(), true);
```


1. 在 TextureMovieEncoder::prepareEncoder 初始化一些实例 
    - 实例化 VideoEncoderCore
    - 实例化 WindowSurface，注意这里 传入了 编码器创建的 Surface，然后里面一通操作将这 Surface 和 OpenGl绑定起来

    ```
    private void prepareEncoder(EGLContext sharedContext, int width, int height, int bitRate,
                                File outputFile) {
        // 视频编码的辅助类，负责配置编码器、处理编码数据以及将编码后的数据写入到输出文件中
        mVideoEncoder = new VideoEncoderCore(width, height, bitRate, outputFile);
        
        // 管理EGL（OpenGL ES上下文）的辅助类，用于在Android中创建和管理EGL上下文、表面和配置
        mEglCore = new EglCore(sharedContext, EglCore.FLAG_RECORDABLE);
        
        // WindowSurface 是 EGLSurface（OpenGL渲染表面）的辅助类，封装了创建、管理和操作 EGLSurface 的方法
        // 注意：传入 编码器创建的 Surface，内部方法会将 Surface 和 OpenGL绑定起来，也就将 openGL 和 MediaCodec绑定起来了
        mInputWindowSurface = new WindowSurface(mEglCore, mVideoEncoder.getInputSurface(), true);
        
        // 将 EGLSurface 和 EGLContext（OpenGL上下文）绑定为当前的渲染目标。
        mInputWindowSurface.makeCurrent();
        
        // 管理 Texture2dProgram 的辅助类 ，Texture2dProgram 是管理 OpenGL ES 着色器程序的类
        mFullScreen = new FullFrameRect(
                new Texture2dProgram(Texture2dProgram.ProgramType.TEXTURE_EXT));
    }
    ```

2. 在 GLSurfaceView 的 onDrawFrame 回调里将 openGL 的纹理ID 传到 TextureMovieEncoder 
    ```
    mVideoEncoder.setTextureId(mTextureId);
    ```
3. 紧接着 TextureMovieEncoder 调用 frameAvailable 传入上面由 纹理ID创建的 mSurfaceTexture ，通知 TextureMovieEncoder 处理可用的帧
    ```
    mVideoEncoder.frameAvailable(mSurfaceTexture);
    ```
    - frameAvailable 本质就是调用了 handleFrameAvailable，两个入参，
        - 参数1 float[] transform: 通过 SurfaceTexture 调用 getTransformMatrix 获取到，包含了相机预览到 SurfaceTexture 的变换矩阵。这个变换矩阵描述了相机预览帧在屏幕上的变换，包括旋转、缩放、平移等操作。确保预览帧以正确的方向和位置呈现在渲染画布上。
        - 参数2 long timestampNanos：通过 SurfaceTexture 获取到帧时间戳，这个是相机填充帧到 SurfaceTexture 确定的时间戳
    ```
    private void handleFrameAvailable(float[] transform, long timestampNanos) {
        // 让编码器尝试进行编码，见预览帧转换为视频编码帧， false 表示不强制完成编码（如果还有缓冲区未编码完成）
        mVideoEncoder.drainEncoder(false);
        // 这里实际调用了 Texture2dProgram::draw ，进行渲染，并将渲染结果保存到纹理ID mTextureId 上面
        mFullScreen.drawFrame(mTextureId, transform);
        // 绘制一个彩色方框
        drawBox(mFrameNum++);
        // 设置输入窗口表面的显示时间，确保视频录制和渲染的同步。
        mInputWindowSurface.setPresentationTime(timestampNanos);
        // 交换输入窗口表面的前后缓冲，将渲染的图像显示在屏幕上。
        mInputWindowSurface.swapBuffers();
    }
    ```




- 编码器设置纹理ID
    handleSetTexture

SurfaceTexture.getTimestamp()


#### 保存渲染视频

1. startEncoder 开始录制
    - 1.1 计算黑边
    - 1.2 实例化几个辅助类
        - 编码辅助类 VideoEncoderCore 
        - 管理 EGLSurface(OpenGL渲染表面) 辅助类 WindowSurface 
        - 用于保存视频的子线程维护类 TextureMovieEncoder2



2. 通过 Choreographer 的 doFrame 回调 和屏幕刷新率保持一致

Choreographer 的 doFrame() 方法触发机制与系统的刷新率有关。在 Android 中，系统会以固定的时间间隔发送 VSync（垂直同步）信号到屏幕上，来指示开始绘制下一帧画面。Choreographer 利用这个信号来同步应用的帧绘制。
Android 的屏幕刷新率是 60Hz，也就是每秒钟会刷新 60 帧。

这里会调用子线程 RenderThread 的 doFrame 函数

---

### API 说明

#### EglSurfaceBase

用于管理 EGLSurface（OpenGL渲染表面）的辅助类，它封装了创建、管理和操作 EGLSurface 的方法，以便在 OpenGL 环境中进行渲染和绘制。


- createWindowSurface(Object surface)：
    创建一个窗口表面，接受一个参数 surface，可以是 Surface 或 SurfaceTexture。它会在 EGL 环境中创建一个与输入 surface 相关联的 EGLSurface。

- createOffscreenSurface(int width, int height)：
    创建一个离屏表面，接受宽度和高度作为参数。它会在 EGL 环境中创建一个离屏的 EGLSurface。

- getWidth() 和 getHeight()：
    返回 EGLSurface 的宽度和高度。

- releaseEglSurface()：
    释放 EGLSurface，清除相关资源。

- makeCurrent()：
    将 EGLSurface 和 EGLContext（OpenGL上下文）绑定为当前的渲染目标。

- makeCurrentReadFrom(EglSurfaceBase readSurface)：
    使当前 EGLSurface 用于绘制，但读取另一个 readSurface 的 EGLSurface 的内容。

- swapBuffers()：
    执行双缓冲区交换，将渲染的内容呈现到屏幕上。

- setPresentationTime(long nsecs)：
    设置 EGLSurface 的呈现时间戳，用于同步渲染和显示的时机。

- saveFrame(File file)：
    将 EGLSurface 的内容保存为一个图像文件，通常用于调试和截屏。

总的来说，EglSurfaceBase 类提供了在 EGL 环境中管理渲染表面的方法，使得 OpenGL 渲染操作更加便捷和可控。它通常作为其他渲染逻辑的基础，用于实现渲染和图像处理的功能。

#### VideoEncoderCore

一个用于视频编码的辅助类，主要负责配置编码器、处理编码数据以及将编码后的数据写入到输出文件中。

- VideoEncoderCore(int width, int height, int bitRate, File outputFile)：
    这个构造函数用于初始化视频编码器。它接受视频的宽度、高度、比特率和输出文件作为参数。在构造函数中，会根据传入的参数配置编码器的参数、创建输入 Surface，以及创建 MediaMuxer 用于写入编码后的数据。

- Surface getInputSurface()：
    这个方法返回编码器的输入 Surface，可以将图像数据渲染到这个 Surface 上，然后交给编码器进行编码。

- void drainEncoder(boolean endOfStream)：
    这个方法用于处理编码后的数据，并将数据写入到输出文件中。它需要传入一个参数 endOfStream，表示是否编码结束。在这个方法中，会不断地从编码器的输出缓冲区中获取编码后的数据，并写入到输出文件中。如果传入的 endOfStream 为 true，则会发送 EOS（End Of Stream）信号给编码器，然后等待编码器输出完成。

    这个方法的核心循环会不断地从编码器的输出缓冲区获取数据，判断是否为关键帧或非关键帧，然后将数据写入到输出文件中。在循环结束时，会释放编码器的输出缓冲区，并判断是否到达文件的末尾。

    这个方法的处理逻辑保证了编码后的数据正确地写入到输出文件中，并且会根据关键帧和非关键帧的情况进行适当的处理。

- void release()：
    这个方法用于释放编码器和输出文件的资源。在这个方法中，会停止编码器，释放相关资源，关闭输出文件。

总之，VideoEncoderCore 类是一个用于视频编码的工具类，封装了编码器的配置、输入 Surface 的创建、编码数据的处理以及输出文件的写入等功能，方便开发者进行视频编码操作。


#### EglCore

一个用于管理EGL（OpenGL ES上下文）的辅助类，用于在Android中创建和管理EGL上下文、表面和配置。

- 构造函数 EglCore(EGLContext sharedContext, int flags)
    构造函数用于初始化EGL显示和上下文。可以通过设置sharedContext来与其他上下文共享资源。flags参数是一些标志，例如FLAG_RECORDABLE（表面必须可记录）和FLAG_TRY_GLES3（尝试使用GLES3，如果不可用则回退到GLES2）。

- createWindowSurface(Object surface)
    创建一个与Surface或SurfaceTexture关联的EGL表面。这是用于渲染的绘制表面。

- createOffscreenSurface(int width, int height)：
    创建一个离屏缓冲的EGL表面。这种表面不会直接显示在屏幕上，通常用于渲染到纹理等其他用途。

- makeCurrent(EGLSurface eglSurface) 
- makeCurrent(EGLSurface drawSurface, EGLSurface readSurface)：
    使指定的EGL表面成为当前的绘制和读取表面。这用于将EGL上下文与指定的表面关联起来，以进行绘制和渲染操作。

- swapBuffers(EGLSurface eglSurface)
    交换指定表面的前后缓冲，将绘制内容显示在屏幕上。

- setPresentationTime(EGLSurface eglSurface, long nsecs)
    设置EGL表面的显示时间戳，以确保视频和音频的同步。

- release()
    释放EGL显示和上下文。应该在不再使用EGL时调用此方法，以确保资源得到释放。

- getGlVersion()
    获取EGL上下文的OpenGL ES版本，可能是2或3。

这个类用于创建和管理EGL上下文，确保在渲染和绘制时进行正确的切换和管理。在上面的代码中，EglCore 被用于创建一个EGL上下文，从而使 TextureMovieEncoder 能够在特定的 EGL 上下文中进行渲染和编码操作。



#### Texture2dProgram

用于管理 OpenGL ES 着色器程序的类。这个类用于创建不同类型的着色器程序，用于绘制不同类型的纹理

- 枚举 ProgramType：
    定义了不同类型的着色器程序，如 TEXTURE_2D、TEXTURE_EXT、TEXTURE_EXT_BW 和 TEXTURE_EXT_FILT。

- 一坨着色器代码：
    包括了一些 OpenGL ES 着色器代码，如顶点着色器和多个片段着色器，用于渲染不同类型的纹理。

- 构造函数：Texture2dProgram(ProgramType programType)
    根据传入的 ProgramType 创建对应类型的着色器程序。

- createTextureObject()：
    创建一个与着色器程序兼容的纹理对象。

- setKernel(float[] values, float colorAdj)：
    用于设置卷积滤波器的参数。

- setTexSize(int width, int height)：
    设置纹理的大小，用于纹理过滤。

- draw(...)：
    用于执行绘制操作，将纹理绘制到屏幕上。

- release()：
    释放着色器程序。

这个类的主要作用是管理和处理不同类型的 OpenGL ES 着色器程序，使得在渲染过程中能够根据需要使用不同的着色器来绘制不同类型的纹理。在你的代码中，Texture2dProgram 类被用于创建和管理着色器程序，用于绘制不同类型的纹理。






