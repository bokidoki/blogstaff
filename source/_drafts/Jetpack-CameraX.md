---
title: Jetpack-CameraX
tags: jetpack
categories: Android
date: 2019/06/09
img: http://dreamweaver.img.we1code.cn/cameraX.png
---

## 前言

今年的Google I/O大会，google发布Jetpack的新成员CameraX，Android Camera Platform team 的产品经理Vinit Modi在开场时说到这个库容易使用并能减少代码量（在视频中，modi提到Camera360使用CameraX减少了75%的代码量），感兴趣的同学可以去看看这个[视频]("https://www.youtube.com/watch?v=kuv8uK-5CLY")。对于这种偷懒的好事我怎么能错过呢，于是赶紧去Github上把[官方demo]("https://github.com/android/camera")clone了下来。项目中使用的CameraX的版本号为1.0.0-alpha02，已经是最新版本了。

```groovy
    def camerax_version = "1.0.0-alpha02"
    implementation "androidx.camera:camera-core:${camerax_version}"
    implementation "androidx.camera:camera-camera2:${camerax_version}"
```

其实camera-camera2是[依赖]("https://android.googlesource.com/platform/frameworks/support/+/ac0c1d466abcfdcab2babb2e10eca574247e3c92/camera/camera2/build.gradle")了camera-core的

## How TO Use

CameraX有三种使用案例，分别是Preview，ImageAnalysis，ImageCapture分别对应着预览、图像分析和拍照，下面看看这三种场景的使用方法：

```koltin
val texture = //initialized
val preview = Preview(PreviewConfig.Builder().build())
preview.setOnPreviewOutputUpdateListener {
    output-> texture.surfaceTexture = output.surfaceTexture
}
CameraX.bindToLifecycle(this, preview)
```

像这样就能够调到相机了，是不是比以前方便很多了？

UseCase 所有Case的抽象类

Set<StateChangeListener> mListeners

Map<String, CameraControl> mAttachedCameraControlMap

Map<String, SessionConfig> mAttachedCameraIdToSessionConfigMap

Map<String, Size> mAttachedSurfaceResolutionMap

State mState = State.INACTIVE; //State.ACTIVE

UseCaseConfig<?> mUseCaseConfig;

int mImageFormat = ImageFormatConstants.INTERNAL_DEFINED_IMAGE_FORMAT_PRIVATE; //INTERNAL_DEFINED_IMAGE_FORMAT_JPEG

CameraX 单例模式

```java
    private static final CameraX INSTANCE = new CameraX();
    final CameraRepository mCameraRepository = new CameraRepository();
    private final AtomicBoolean mInitialized = new AtomicBoolean(false);
    private final UseCaseGroupRepository mUseCaseGroupRepository = new UseCaseGroupRepository();
    private final ErrorHandler mErrorHandler = new ErrorHandler();
    private CameraFactory mCameraFactory;
    private CameraDeviceSurfaceManager mSurfaceManager;
    private UseCaseConfigFactory mDefaultConfigFactory;
    private Context mContext;

    bindToLifecycle()

```

cameraX初始化在Camera2Initializer下，我们不需要通过application去初始化，因为Camera2Initializer继承了ContentProvider，Camera2Initializer位于camera-camera2包中，在onCreate()中做了初始化Camera2AppConfig的工作，Camera2AppConfig是如何被创建的呢？在Camera2AppConfig.create中

LensFacing相机相对于屏幕的方向，FRONT，BACK

```java
    public static AppConfig create(Context context) {
        // Create the camera factory for creating Camera2 camera objects
        CameraFactory cameraFactory = new Camera2CameraFactory(context);

        // Create the DeviceSurfaceManager for Camera2
        CameraDeviceSurfaceManager surfaceManager = new Camera2DeviceSurfaceManager(context);

        // Create default configuration factory
        ExtendableUseCaseConfigFactory configFactory = new ExtendableUseCaseConfigFactory();
        configFactory.installDefaultProvider(
                ImageAnalysisConfig.class, new ImageAnalysisConfigProvider(cameraFactory, context));
        configFactory.installDefaultProvider(
                ImageCaptureConfig.class, new ImageCaptureConfigProvider(cameraFactory, context));
        configFactory.installDefaultProvider(
                VideoCaptureConfig.class, new VideoCaptureConfigProvider(cameraFactory, context));
        configFactory.installDefaultProvider(
                PreviewConfig.class, new PreviewConfigProvider(cameraFactory, context));

        AppConfig.Builder appConfigBuilder =
                new AppConfig.Builder()
                        .setCameraFactory(cameraFactory)
                        .setDeviceSurfaceManager(surfaceManager)
                        .setUseCaseConfigFactory(configFactory);

        return appConfigBuilder.build();
    }
```

创建了Camera2CameraFactory主要提供getCamera、getAvailableCameraIds、cameraIdForLensFacing，在Camera2CameraFactory的构造方法中我们看到了熟悉的代码：

```java
mCameraManager = (CameraManager) context.getSystemService(Context.CAMERA_SERVICE);
```

在getCamera中我们得到了Camera对象，它对相机的基本操作做了封装
接着往下看ExtendableUseCaseConfigFactory是camera-core的一个类，用于收集各种UseCase的ConfigProvider
