# Android Training 2

## Android分享操作

### 给其他App发送简单的数据

为了发送数据到另外一个activity，我们只需要指定数据与数据的类型，系统会自动识别出能够兼容接受的这些数据的activity。如果这些选择有多个，则把这些activity显示给用户进行选择；如果只有一个，则立即启动该Activity。同样的，我们可以在manifest文件的Activity描述中添加接受的数据类型。

如果为intent调用了Intent.createChooser()，那么Android总是会显示可供选择。这样有一些好处：

* 即使用户之前为这个intent设置了默认的action，选择界面还是会被显示。
* 如果没有匹配的程序，Android会显示系统信息。
* 我们可以指定选择界面的标题。

#### 分享二进制内容


```
Intent shareIntent = new Intent();
shareIntent.setAction(Intent.ACTION_SEND);
shareIntent.putExtra(Intent.EXTRA_STREAM, uriToImage);
shareIntent.setType("image/jpeg");
startActivity(Intent.
```
请注意以下内容：

* 我们可以使用*/*这样的方式来指定MIME类型，但是这仅仅会match到那些能够处理一般数据类型的Activity(即一般的Activity无法详尽所有的MIME类型)
* 接收的程序需要有访问URI资源的权限。下面有一些方法来处理这个问题：
	* 将数据存储在ContentProvider中，确保其他程序有访问provider的权限。较好的提供访问权限的方法是使用 per-URI permissions，其对接收程序而言是只是暂时拥有该许可权限。类似于这样创建ContentProvider的一种简单的方法是使用FileProvider helper类。
	* 使用MediaStore系统。MediaStore系统主要用于音视频及图片的MIME类型。但在Android3.0之后，其也可以用于存储非多媒体类型。

#### 发送多块内容

为了同时分享多种不同类型的内容，需要使用Intent.ACTION_SEND_MULTIPLE与指定到那些数据的URIs列表。


### 接收从其他App传送来的数据

#### 更新我们的manifest文件

创建intent filters来表明程序能够接收的action类型。下面是个例子，对三个activit分别指定接受单张图片，文本与多张图片。

```
<activity android:name=".ui.MyActivity" >
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="image/*" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.SEND_MULTIPLE" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="image/*" />
    </intent-filter>
</activity>
```

#### 处理接受到的数据

请记住，如果一个activity可以被其他的程序启动，我们需要在检查intent的时候考虑这种情况(是被其他程序而调用启动的)。

请注意，由于无法知道其他程序发送过来的数据内容是文本还是其他类型的数据，若数据量巨大，则需要大量处理时间，因此我们应避免在UI线程里面去处理那些获取到的数据。

### 分享文件

#### 建立文件分享

为了将文件安全地从我们的应用程序共享给其它应用程序，我们需要对自己的应用进行配置来提供安全的文件句柄（Content URI的形式）。Android的FileProvider组件会基于在XML文件中的具体配置为文件创建Content URI。

为了给应用程序定义一个FileProvider，需要在Manifest清单文件中定义一个entry，该entry指明了需要使用的创建Content URI的Authority。此外，还需要一个XML文件的文件名，该XML文件指定了我们的应用可以共享的目录路径。

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.myapp">
    <application
        ...>
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="com.example.myapp.fileprovider"
            android:grantUriPermissions="true"
            android:exported="false">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/filepaths" />
        </provider>
        ...
    </application>
</manifest>
```

对于自己的应用，要在我们的应用程序包名（android:package的值）之后继续追加“fileprovider”来指定Authority

res/xml/filepaths.xml内容样例。内部存储区域共享一个“files/”目录的子目录：

```
<paths>
    <files-path path="images/" name="myimages" />
</paths>
```

<paths>标签可以有多个子标签，每一个子标签用来指定不同的共享目录。除了<files-path>标签，还可以使用<external-path>来共享位于外部存储的目录；另外，<cache-path>标签用来共享在内部缓存目录下的子目录。

**Note: XML文件是我们定义共享目录的唯一方式，不可以用代码的形式添加目录。**

例如，如果本课的例子定义了一个FileProvider，然后我们需要一个文件“default_image.jpg”的Content URI，FileProvider会返回如下URI：

```
content://com.example.myapp.fileprovider/myimages/default_image.jpg

```

#### 为文件授权

现在已经有了想要共享给其他应用程序的文件所对应的Content URI，我们需要允许客户端应用程序访问这个文件。为了达到这一目的，可以通过将Content URI添加至一个Intent中，然后为该Intent设置权限标记。所授予的权限是临时的，并且当接收文件的应用程序的任务栈终止后，会自动过期。


```
mResultIntent.addFlags(
                        Intent.FLAG_GRANT_READ_URI_PERMISSION);
```
调用setFlags()来为文件授予临时被访问权限是唯一的安全的方法。尽量避免对文件的Content URI调用Context.grantUriPermission()，因为通过该方法授予的权限，只能通过调用Context.revokeUriPermission()来撤销。

#### 访问请求的文件

这一过程中不用过多担心文件的安全问题，因为客户端应用程序所收到的所有数据只有文件的Content URI而已。由于URI不包含目录路径信息，客户端应用程序无法查询或打开任何服务端应用程序的其他文件。客户端应用程序仅仅获取了这个文件的访问渠道以及由服务端应用程序授予的访问权限。同时访问权限是临时的，一旦这个客户端应用的任务栈结束了，这个文件将无法再被除服务端应用程序之外的其他应用程序访问。


```

    @Override
    public void onActivityResult(int requestCode, int resultCode,
            Intent returnIntent) {
        if (resultCode != RESULT_OK) {
            return;
        } else {
            Uri returnUri = returnIntent.getData();
            try {
                mInputPFD = getContentResolver().openFileDescriptor(returnUri, "r");
            } catch (FileNotFoundException e) {
                e.printStackTrace();
                Log.e("MainActivity", "File not found.");
                return;
            }
            FileDescriptor fd = mInputPFD.getFileDescriptor();
            ...
        }
    }
```
openFileDescriptor()方法返回一个文件的ParcelFileDescriptor对象。客户端应用程序从该对象中获取FileDescriptor对象，然后利用该对象读取这个文件了。

### 使用NFC分享文件

使用Android Beam文件传输功能必须满足以下要求：

* Android Beam文件传输功能传输大文件必须在Android 4.1（API Level 16）及以上版本的Android系统中使用。
* 希望传送的文件必须放置于外部存储。更多关于外部存储的知识，请参考：Using the External Storage。
* 希望传送的文件必须是全局可读的。我们可以通过File.setReadable(true,false)来为文件设置相应的读权限。
* 必须提供待传输文件的File URI。Android Beam文件传输无法处理由FileProvider.getUriForFile生成的Content URI。

#### 在清单文件中声明

NFC

```
<uses-permission android:name="android.permission.NFC" />
```
READ_EXTERNAL_STORAGE

```
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
```

#### 指定NFC功能

设置android:required属性字段为true，使得我们的应用程序只有在NFC可以使用时才能运行。

```
<uses-feature android:name="android.hardware.nfc"
    android:required="true" />
```
注意，如果应用程序将NFC作为一个可选的功能，期望在NFC不可使用时程序还能继续执行，我们就应该将android:required属性字段设为false，然后在代码中测试NFC的可用性。

#### 测试设备是否支持Android Beam文件传输


```
	// NFC isn't available on the device
	if (!PackageManager.hasSystemFeature(PackageManager.FEATURE_NFC)) {
		...
	// Android Beam file transfer isn't supported
    } else if (Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR1) {
    	mAndroidBeamAvailable = false;
		...
    // Android Beam file transfer is available, continue
    } else {
        mNfcAdapter = NfcAdapter.getDefaultAdapter(this);
        ...
    }
```

#### 从File URI中获取目录

如果接收的Intent包含一个File URI，则该URI包含了一个文件的绝对文件名，它包括了完整的路径和文件名。
需要取得URI的路径部分（URI中除去“file:”前缀的部分）
```
...
    public String handleFileUri(Uri beamUri) {
        // Get the path part of the URI
        String fileName = beamUri.getPath();
        // Create a File object for this filename
        File copiedFile = new File(fileName);
        // Get a string containing the file's parent directory
        return copiedFile.getParent();
    }
    ...
```

#### 从Content URI获取目录

为了确定是否能从Content URI中获取文件目录，可以通过调用Uri.getAuthority()获取URI的Authority，以此确定与该URI相关联的Content Provider。

其结果有两个可能的值：

* MediaStore.AUTHORITY

表明该URI关联了被MediaStore记录的一个文件或者多个文件。可以从MediaStore中获取文件的全名，目录名就自然可以从文件全名中获取。

* 其他值

来自其他Content Provider的Content URI。可以显示与该Content URI相关联的数据，但是不要尝试去获取文件目录。

要从MediaStore的Content URI中获取目录，我们需要执行一个查询操作，它将Uri参数指定为收到的ContentURI，将MediaColumns.DATA列作为投影（Projection）。返回的Cursor对象包含了URI所代表的文件的完整路径和文件名。


```
    ...
    public String handleContentUri(Uri beamUri) {
        // Position of the filename in the query Cursor
        int filenameIndex;
        // File object for the filename
        File copiedFile;
        // The filename stored in MediaStore
        String fileName;
        // Test the authority of the URI
        if (!TextUtils.equals(beamUri.getAuthority(), MediaStore.AUTHORITY)) {
            /*
             * Handle content URIs for other content providers
             */
        // For a MediaStore content URI
        } else {
            // Get the column that contains the file name
            String[] projection = { MediaStore.MediaColumns.DATA };
            Cursor pathCursor =
                    getContentResolver().query(beamUri, projection,
                    null, null, null);
            // Check for a valid cursor
            if (pathCursor != null &&
                    pathCursor.moveToFirst()) {
                // Get the column index in the Cursor
                filenameIndex = pathCursor.getColumnIndex(
                        MediaStore.MediaColumns.DATA);
                // Get the full file name including path
                fileName = pathCursor.getString(filenameIndex);
                // Create a File object for the filename
                copiedFile = new File(fileName);
                // Return the parent directory of the file
                return new File(copiedFile.getParent());
             } else {
                // The query didn't work; return null
                return null;
             }
        }
    }
    ...
```


