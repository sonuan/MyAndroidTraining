# # Android Training 3

## 管理音频播放

### 控制音量与音频播放(Controlling Your App’s Volume and Playback)

独立的音频流：播放音乐，闹铃，通知铃，来电声音，系统声音，打电话声音与拨号声音。主要目的是让用户能够单独地控制不同的种类的音频。

默认情况下，按下音量控制键会调节当前被激活的音频流，如果我们的应用当前没有播放任何声音，那么按下音量键会调节响铃的音量。Android提供了setVolumeControlStream()方法来直接控制指定的音频流。通常，我们可以在负责控制多媒体的Activity或者Fragment的onCreate()方法中调用它。这样能确保不管应用当前是否可见，音频控制的功能都能符合用户的预期。

#### 使用硬件的播放控制按键来控制应用的音频播放(Use Hardware Playback Control Keys to Control Your App’s Audio Playback)
许多线控或者无线耳机都会有许多媒体播放控制按钮，例如：播放，停止，暂停，跳过，以及回放等。无论用户按下设备上任意一个控制按钮，系统都会广播一个带有ACTION_MEDIA_BUTTON的Intent。Intent通过EXTRA_KEY_EVENT这一Key包含了该信息，另外，KeyEvent类包含了一系列诸如KEYCODE_MEDIA_*的静态变量来表示不同的媒体按钮，例如KEYCODE_MEDIA_PLAY_PAUSE 与 KEYCODE_MEDIA_NEXT。

因为可能会有多个程序在监听与媒体按钮相关的事件，所以我们必须在代码中控制应用接收相关事件的时机。

```
AudioManager am = mContext.getSystemService(Context.AUDIO_SERVICE);
...

// Start listening for button presses
am.registerMediaButtonEventReceiver(RemoteControlReceiver);
...

// Stop listening for button presses
am.unregisterMediaButtonEventReceiver(RemoteControlReceiver);
```

### 管理音频焦点(Managing Audio Focus)

#### 请求获取音频焦点(Request the Audio Focus)


```
AudioManager am = mContext.getSystemService(Context.AUDIO_SERVICE);
...

// Request audio focus for playback
int result = am.requestAudioFocus(afChangeListener,
                                 // Use the music stream.
                                 AudioManager.STREAM_MUSIC,
                                 // Request permanent focus.
                                 AudioManager.AUDIOFOCUS_GAIN);

if (result == AudioManager.AUDIOFOCUS_REQUEST_GRANTED) {
    am.registerMediaButtonEventReceiver(RemoteControlReceiver);
    // Start playback.
}
```
一旦结束了播放，需要确保调用了abandonAudioFocus()方法。这样相当于告知系统我们不再需要获取焦点并且注销所关联的AudioManager.OnAudioFocusChangeListener监听器。

当请求短暂音频焦点的时候，我们可以选择是否开启“Ducking”。通常情况下，一个应用在失去音频焦点时会立即关闭它的播放声音。如果我们选择在请求短暂音频焦点的时候开启了Ducking，那意味着其它应用可以继续播放，仅仅是在这一刻降低自己的音量，直到重新获取到音频焦点后恢复正常音量


```
// Request audio focus for playback
int result = am.requestAudioFocus(afChangeListener,
                             // Use the music stream.
                             AudioManager.STREAM_MUSIC,
                             // Request permanent focus.
                             AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK);

if (result == AudioManager.AUDIOFOCUS_REQUEST_GRANTED) {
    // Start playback.
}
```
Ducking对于那些间歇性使用音频焦点的应用来说特别合适，比如语音导航。

#### 处理失去音频焦点(Handle the Loss of Audio Focus)

在音频焦点的监听器里面，当接受到描述焦点改变的事件时会触发onAudioFocusChange()回调方法。三种失去焦点的类型：永久失去，短暂失去，允许Ducking的短暂失去。

* 失去短暂焦点：通常在失去短暂焦点的情况下，我们会暂停当前音频的播放或者降低音量，同时需要准备在重新获取到焦点之后恢复播放。

* 失去永久焦点：假设另外一个应用开始播放音乐，那么我们的应用就应该有效地将自己停止。在实际场景当中，这意味着停止播放，移除媒体按钮监听，允许新的音频播放器可以唯一地监听那些按钮事件，并且放弃自己的音频焦点。此时，如果想要恢复自己的音频播放，我们需要等待某种特定用户行为发生（例如按下了我们应用当中的播放按钮）。


```
OnAudioFocusChangeListener afChangeListener = new OnAudioFocusChangeListener() {
    public void onAudioFocusChange(int focusChange) {
        if (focusChange == AUDIOFOCUS_LOSS_TRANSIENT
            // Pause playback
        } else if (focusChange == AudioManager.AUDIOFOCUS_GAIN) {
            // Resume playback 
        } else if (focusChange == AudioManager.AUDIOFOCUS_LOSS) {
            am.unregisterMediaButtonEventReceiver(RemoteControlReceiver);
            am.abandonAudioFocus(afChangeListener);
            // Stop playback
        }else if (focusChange == AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK) { // Duck!
            // Lower the volume
        } else if (focusChange == AudioManager.AUDIOFOCUS_GAIN) { // Duck!
            // Raise it back to normal
        }
    }
};
```


### 兼容音频输出设备(Dealing with Audio Output Hardware)

#### 检测目前正在使用的硬件设备(Check What Hardware is Being Used)

可以使用AudioManager来查询当前音频是输出到扬声器，有线耳机还是蓝牙上，如下所示：


```
if (isBluetoothA2dpOn()) {
    // Adjust output for Bluetooth.
} else if (isSpeakerphoneOn()) {
    // Adjust output for Speakerphone.
} else if (isWiredHeadsetOn()) {
    // Adjust output for headsets
} else { 
    // If audio plays and noone can hear it, is it still playing?
}
```


#### 处理音频输出设备的改变(Handle Changes in the Audio Output Hardware)
当有线耳机被拔出或者蓝牙设备断开连接的时候，音频流会自动输出到内置的扬声器上。假设播放声音很大，这个时候突然转到扬声器播放会显得非常嘈杂。幸运的是，系统会在这种情况下广播带有ACTION_AUDIO_BECOMING_NOISY的Intent。无论何时播放音频，我们都应该注册一个BroadcastReceiver来监听这个Intent。

## 拍照

### 简单的拍照

#### 请求使用相机权限


```
<manifest ... >
    <uses-feature android:name="android.hardware.camera"
                  android:required="true" />
    ...
</manifest>
```
如果我们的应用使用相机，但相机并不是应用的正常运行所必不可少的组件，可以将android:required设置为"false"。这样的话，Google Play 也会允许没有相机的设备下载该应用。当然我们有必要在使用相机之前通过调用hasSystemFeature(PackageManager.FEATURE_CAMERA)方法来检查设备上是否有相机。如果没有，我们应该禁用和相机相关的功能！

#### 使用相机应用程序进行拍照


```
static final int REQUEST_IMAGE_CAPTURE = 1;

private void dispatchTakePictureIntent() {
    Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    if (takePictureIntent.resolveActivity(getPackageManager()) != null) {
        startActivityForResult(takePictureIntent, REQUEST_IMAGE_CAPTURE);
    }
}
```

先调用resolveActivity()，即检查有没有能处理这个Intent的Activity。执行这个检查非常重要，因为如果在调用startActivityForResult()时，没有应用能处理你的Intent，应用将会崩溃。所以只要返回结果不为null，使用该Intent就是安全的。

#### 获取缩略图


```
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
        Bundle extras = data.getExtras();
        Bitmap imageBitmap = (Bitmap) extras.get("data");
        mImageView.setImageBitmap(imageBitmap);
    }
}
```

这张从"data"中取出的缩略图适用于作为图标，但其他作用会比较有限。而处理一张全尺寸图片需要做更多的工作。

#### 保存全尺寸照片


```
static final int REQUEST_TAKE_PHOTO = 1;

private void dispatchTakePictureIntent() {
    Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    // Ensure that there's a camera activity to handle the intent
    if (takePictureIntent.resolveActivity(getPackageManager()) != null) {
        // Create the File where the photo should go
        File photoFile = null;
        try {
            photoFile = createImageFile();
        } catch (IOException ex) {
            // Error occurred while creating the File
            ...
        }
        // Continue only if the File was successfully created
        if (photoFile != null) {
            takePictureIntent.putExtra(MediaStore.EXTRA_OUTPUT,
                    Uri.fromFile(photoFile));
            startActivityForResult(takePictureIntent, REQUEST_TAKE_PHOTO);
        }
    }
}
```

#### 将照片添加到相册中

由于我们通过Intent创建了一张照片，因此图片的存储位置我们是知道的。对其他人来说，也许查看我们的照片最简单的方式是通过系统的Media Provider。 
如果将图片存储在getExternalFilesDir()提供的目录中，Media Scanner将无法访问到我们的文件，因为它们隶属于应用的私有数据。


```
private void galleryAddPic() {
    Intent mediaScanIntent = new Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE);
    File f = new File(mCurrentPhotoPath);
    Uri contentUri = Uri.fromFile(f);
    mediaScanIntent.setData(contentUri);
    this.sendBroadcast(mediaScanIntent);
}
```

#### 解码一幅缩放图片


```
private void setPic() {
    // Get the dimensions of the View
    int targetW = mImageView.getWidth();
    int targetH = mImageView.getHeight();

    // Get the dimensions of the bitmap
    BitmapFactory.Options bmOptions = new BitmapFactory.Options();
    bmOptions.inJustDecodeBounds = true;
    BitmapFactory.decodeFile(mCurrentPhotoPath, bmOptions);
    int photoW = bmOptions.outWidth;
    int photoH = bmOptions.outHeight;

    // Determine how much to scale down the image
    int scaleFactor = Math.min(photoW/targetW, photoH/targetH);

    // Decode the image file into a Bitmap sized to fill the View
    bmOptions.inJustDecodeBounds = false;
    bmOptions.inSampleSize = scaleFactor;
    bmOptions.inPurgeable = true;

    Bitmap bitmap = BitmapFactory.decodeFile(mCurrentPhotoPath, bmOptions);
    mImageView.setImageBitmap(bitmap);
}
```

### 简单的录像


```
static final int REQUEST_VIDEO_CAPTURE = 1;

private void dispatchTakeVideoIntent() {
    Intent takeVideoIntent = new Intent(MediaStore.ACTION_VIDEO_CAPTURE);
    if (takeVideoIntent.resolveActivity(getPackageManager()) != null) {
        startActivityForResult(takeVideoIntent, REQUEST_VIDEO_CAPTURE);
    }
}
```

```@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_VIDEO_CAPTURE && resultCode == RESULT_OK) {
        Uri videoUri = intent.getData();
        mVideoView.setVideoURI(videoUri);
    }
}
```

### 控制相机

#### 打开相机对象

比较好的访问相机的方式是在onCreate()方法里面另起一个线程来打开相机。这种办法可以避免因为启动时间较长导致UI线程被阻塞。另外还有一种更好的方法：可以把打开相机的操作延迟到onResume()方法里面去执行，这样可以使得代码更容易重用，还能保持控制流程更为简单。

如果我们在执行Camera.open()方法的时候相机正在被另外一个应用使用，那么函数会抛出一个Exception，我们可以利用try语句块进行捕获：


```
private boolean safeCameraOpen(int id) {
    boolean qOpened = false;

    try {
        releaseCameraAndPreview();
        mCamera = Camera.open(id);
        qOpened = (mCamera != null);
    } catch (Exception e) {
        Log.e(getString(R.string.app_name), "failed to open Camera");
        e.printStackTrace();
    }

    return qOpened;    
}

private void releaseCameraAndPreview() {
    mPreview.setCamera(null);
    if (mCamera != null) {
        mCamera.release();
        mCamera = null;
    }
}
```

#### 创建相机预览界面

Preview类


```
class Preview extends ViewGroup implements SurfaceHolder.Callback {

    SurfaceView mSurfaceView;
    SurfaceHolder mHolder;

    Preview(Context context) {
        super(context);

        mSurfaceView = new SurfaceView(context);
        addView(mSurfaceView);

        // Install a SurfaceHolder.Callback so we get notified when the
        // underlying surface is created and destroyed.
        mHolder = mSurfaceView.getHolder();
        mHolder.addCallback(this);
        mHolder.setType(SurfaceHolder.SURFACE_TYPE_PUSH_BUFFERS);
    }
...
}
```

设置和启动Preview


```
public void setCamera(Camera camera) {
    if (mCamera == camera) { return; }

    stopPreviewAndFreeCamera();

    mCamera = camera;

    if (mCamera != null) {
        List<Size> localSizes = mCamera.getParameters().getSupportedPreviewSizes();
        mSupportedPreviewSizes = localSizes;
        requestLayout();

        try {
            mCamera.setPreviewDisplay(mHolder);
        } catch (IOException e) {
            e.printStackTrace();
        }

        // Important: Call startPreview() to start updating the preview
        // surface. Preview must be started before you can take a picture.
        mCamera.startPreview();
    }
}
```

#### 修改相机设置


```
public void surfaceChanged(SurfaceHolder holder, int format, int w, int h) {
    // Now that the size is known, set up the camera parameters and begin
    // the preview.
    Camera.Parameters parameters = mCamera.getParameters();
    parameters.setPreviewSize(mPreviewSize.width, mPreviewSize.height);
    requestLayout();
    mCamera.setParameters(parameters);

    // Important: Call startPreview() to start updating the preview surface.
    // Preview must be started before you can take a picture.
    mCamera.startPreview();
}
```

#### 设置预览方向

setCameraDisplayOrientation()。对于Android API Level 14及以下版本的系统，在改变方向之前，我们必须先停止预览，然后再去重启它。

#### 拍摄照片
只要预览开始之后，可以使用Camera.takePicture()方法拍摄照片。

如果我们想要进行连拍，可以创建一个Camera.PreviewCallback并实现onPreviewFrame()方法。我们可以拍摄选中的预览帧，或是为调用takePicture()建立一个延迟。

#### 重启Preview


```
@Override
public void onClick(View v) {
    switch(mPreviewState) {
    case K_STATE_FROZEN:
        mCamera.startPreview();
        mPreviewState = K_STATE_PREVIEW;
        break;

    default:
        mCamera.takePicture( null, rawCallback, null);
        mPreviewState = K_STATE_BUSY;
    } // switch
    shutterBtnConfig();
}
```

#### 停止预览并释放相机


```
public void surfaceDestroyed(SurfaceHolder holder) {
    // Surface will be destroyed when we return, so stop the preview.
    if (mCamera != null) {
        // Call stopPreview() to stop updating the preview surface.
        mCamera.stopPreview();
    }
}

/**
 * When this function returns, mCamera will be null.
 */
private void stopPreviewAndFreeCamera() {

    if (mCamera != null) {
        // Call stopPreview() to stop updating the preview surface.
        mCamera.stopPreview();

        // Important: Call release() to release the camera for use by other
        // applications. Applications should release the camera immediately
        // during onPause() and re-open() it during onResume()).
        mCamera.release();

        mCamera = null;
    }
}
```

### 打印照片

#### 打印一幅图片

Android Support Library中的PrintHelper类提供了一种打印图片的简单方法。布局选项：setScaleMode()。下面的两个选项之一：

* SCALE_MODE_FIT：该选项会调整图像的大小，这样整个图像就会在打印有效区域内全部显示出来（等比例缩放至长和宽都包含在纸张页面内）。
* SCALE_MODE_FILL：该选项同样会等比例地调整图像的大小使图像充满整个打印有效区域，即让图像充满整个纸张页面。这就意味着如果选择这个选项，那么图片的一部分（顶部和底部，或者左侧和右侧）将无法打印出来。如果不设置图像的打印布局选项，该模式将是默认的图像拉伸方式。


```
private void doPhotoPrint() {
    PrintHelper photoPrinter = new PrintHelper(getActivity());
    photoPrinter.setScaleMode(PrintHelper.SCALE_MODE_FIT);
    Bitmap bitmap = BitmapFactory.decodeResource(getResources(),
            R.drawable.droids);
    photoPrinter.printBitmap("droids.jpg - test print", bitmap);
}
```

在printBitmap()被调用之后，我们的应用就不再需要进行其他的操作了。之后Android打印界面就会出现，允许用户选择一个打印机和它的打印选项。


#### 打印HTML文档（WebView）


```
private void createWebPrintJob(WebView webView) {

    // Get a PrintManager instance
    PrintManager printManager = (PrintManager) getActivity()
            .getSystemService(Context.PRINT_SERVICE);

    // Get a print adapter instance
    PrintDocumentAdapter printAdapter = webView.createPrintDocumentAdapter();

    // Create a print job with name and adapter instance
    String jobName = getString(R.string.app_name) + " Document";
    PrintJob printJob = printManager.print(jobName, printAdapter,
            new PrintAttributes.Builder().build());

    // Save the job object for later status checking
    mPrintJobs.add(printJob);
}
```

请确保在WebViewClient)中的onPageFinished()方法内调用创建打印任务的方法。如果没有等到页面加载完毕就进行打印，打印的输出可能会不完整或空白，甚至可能会失败。

当使用WebView创建打印文档时，你要注意下面的一些限制：

* 不能为文档添加页眉和页脚，包括页号。
* HTML文档的打印选项不包含选择打印的页数范围，例如：对于一个10页的HTMl文档，只打印2到4页是不可以的。
* 一个WebView的实例只能在同一时间处理一个打印任务。
* 若一个HTML文档包含CSS打印属性，比如一个landscape属性，这是不被支持的。
* 不能通过一个HTML文档中的JavaScript脚本来激活打印。

#### 打印自定义文档


```
private void doPrint() {
    // Get a PrintManager instance
    PrintManager printManager = (PrintManager) getActivity()
            .getSystemService(Context.PRINT_SERVICE);

    // Set job name, which will be displayed in the print queue
    String jobName = getActivity().getString(R.string.app_name) + " Document";

    // Start a print job, passing in a PrintDocumentAdapter implementation
    // to handle the generation of a print document
    printManager.print(jobName, new MyPrintDocumentAdapter(getActivity()),
            null); //
}
```

PrintDocumentAdapter抽象类负责处理打印的生命周期，它有四个主要的回调方法。我们必须在打印适配器中实现这些方法，以此来正确地和Android打印框架进行交互：

* onStart()：一旦打印进程开始，该方法就将被调用。如果我们的应用有任何一次性的准备任务要执行，比如获取一个要打印数据的快照，那么让它们在此处执行。在你的适配器中，这个回调方法不是必须实现的。
* onLayout()：每当用户改变了影响打印输出的设置时（比如改变了页面的尺寸，或者页面的方向）该函数将会被调用，以此给我们的应用一个机会去重新计算打印页面的布局。另外，该方法必须返回打印文档包含多少页面。
* onWrite()：该方法调用后，会将打印页面渲染成一个待打印的文件。该方法可以在onLayout()方法被调用后调用一次或多次。
* onFinish()：一旦打印进程结束后，该方法将会被调用。如果我们的应用有任何一次性销毁任务要执行，让这些任务在该方法内执行。这个回调方法不是必须实现的。

**注意必要时放在线程**

```
@Override
public void onLayout(PrintAttributes oldAttributes,
                     PrintAttributes newAttributes,
                     CancellationSignal cancellationSignal,
                     LayoutResultCallback callback,
                     Bundle metadata) {
    // Create a new PdfDocument with the requested page attributes
    mPdfDocument = new PrintedPdfDocument(getActivity(), newAttributes);

    // Respond to cancellation request
    if (cancellationSignal.isCancelled() ) {
        callback.onLayoutCancelled();
        return;
    }

    // Compute the expected number of printed pages
    int pages = computePageCount(newAttributes);

    if (pages > 0) {
        // Return print information to print framework
        PrintDocumentInfo info = new PrintDocumentInfo
                .Builder("print_output.pdf")
                .setContentType(PrintDocumentInfo.CONTENT_TYPE_DOCUMENT)
                .setPageCount(pages);
                .build();
        // Content layout reflow is complete
        callback.onLayoutFinished(info, true);
    } else {
        // Otherwise report an error to the print framework
        callback.onLayoutFailed("Page count calculation failed.");
    }
}
```
注意：onLayoutFinished()方法的布尔类型参数明确了这个布局内容是否和上一次打印请求相比发生了改变。恰当地设定了这个参数将避免打印框架不必要地调用onWrite()方法，缓存之前的打印文档，提升执行性能。

#### 将打印文档写入文件

当需要将打印内容输出到一个文件时，Android打印框架会调用PrintDocumentAdapter类的onWrite()方法。这个方法的参数指定了哪些页面要被写入以及要使用的输出文件。该方法的实现必须将每一个请求页的内容渲染成一个含有多个页面的PDF文件。当这个过程结束以后，你需要调用callback对象的onWriteFinished()方法。

onWrite()方法的执行可以有三种结果：完成，取消或者失败（内容无法被写入）。我们必须通过调用PrintDocumentAdapter.WriteResultCallback对象中的适当方法来指明这些结果中的一个。



