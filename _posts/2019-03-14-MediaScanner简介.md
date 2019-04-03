---
layout:     post
title:      MediaScanner简介
subtitle:   针对Android的MediaScanner做简单的介绍
date:       2019-02-26
author:     Blank Ting
header-img: img/in-post/post-bg-android.jpg
catalog: 	 true
tags:
    - Android
    - Framework
    - MediaProvider
    - MediaScanner
---

# MediaScanner介绍

## 简介

### 定义

MediaScanner是Android系统中针对媒体文件的扫描过程，将储存空间中的媒体文件通过扫描的方式遍历并存储在数据库中，然后通过MediaProvider提供接口使用，在Android多媒体中占有很重要的位置。

## 扫描触发的方式

- 系统启动
- 媒体文件增加或者删除
- 第三方应用接口调用。

### 场景

音乐，视频播放器，图库等应用关于音视频图片等信息都是通过多媒体数据库直接查询 。我们项目中常会遇到MediaScanner的问题导致文件管理器、闹钟、来电铃声的bug，根本原因就是媒体文件对应的URL错乱或者找不到。这种问题的切入口就是导出MediaProvide的数据库，查看文件的URL。

## MediaScanner的整体结构

## 代码分布

- packages\providers\MediaProvider
- frameworks\base\media\

前者作为一个单独的package，用来接收扫描广播和操作contentProvider、调用扫描服务接口完成扫描；后者中包含了MediaScanner的 jni 和 Java 文件，扫描的大部分工作都在这里。

### MediaProvider代码

- com.android.providers.media.MediaProvider
- com.android.providers.media.MediaDocumentsProvider
- com.android.providers.media.MediaScannerReceiver
- com.android.providers.media.MediaUpgradeReceiver
- com.android.providers.media.MediaScannerService
- com.android.providers.media.MediaThumbRequest
- com.android.providers.media.IMtpService
- com.android.providers.media.MtpReceiver
- com.android.providers.media.MtpService
- com.android.providers.media.RingtonePickerActivity

### framework层代码

- frameworks\base\media\java\android\media
- frameworks\base\media\jni

## 功能分析

关于媒体文件扫描，我们需要弄明白两个问题： 
1.什么时候开启媒体文件扫描 
2.如何解析媒体文件(音频,视频,图片)信息插入到数据库中
MediaScanner的code流程：

![MeidaScanner_code_flow](https://blankting.github.io/img/in-post/mediascanner_code_flow.jpg)

### 应用

#### 开机时触发

##### MediaScannerReceiver

在AndroidManifest中的声明：

``` xml
<receiver android:name="MediaScannerReceiver">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.MEDIA_MOUNTED" />
        <data android:scheme="file" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.MEDIA_UNMOUNTED" />
        <data android:scheme="file" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.MEDIA_SCANNER_SCAN_FILE" />         
        <data android:scheme="file" />
    </intent-filter>
</receiver>
```

启动扫描动作的是BOOT_COMPLETED(开机)，MEDIA_MOUNTED(挂载) MEDIA_UNMOUNTED(卸载) MEDIA_SCANNER_SCAN_FILE(扫描文件)会执行扫描.
看下onReceive中的代码：

``` java
public void onReceive(Context context, Intent intent) {
    final String action = intent.getAction();
    final Uri uri = intent.getData();
    if (Intent.ACTION_BOOT_COMPLETED.equals(action)) {
        // Scan internal only.
        scan(context, MediaProvider.INTERNAL_VOLUME);
    } else if (Intent.ACTION_LOCALE_CHANGED.equals(action)) {
        scanTranslatable(context);
    } else {
        if (uri.getScheme().equals("file")) {
            // handle intents related to external storage
            String path = uri.getPath();
            String externalStoragePath = Environment.getExternalStorageDirectory().getPath();
            String legacyPath = Environment.getLegacyExternalStorageDirectory().getPath();
            
            try {
                path = new File(path).getCanonicalPath();
            } catch (IOException e) {
                Log.e(TAG, "couldn't canonicalize " + path);
                return;
            }
            if (path.startsWith(legacyPath)) {
                path = externalStoragePath + path.substring(legacyPath.length());
            }
            
            Log.d(TAG, "action: " + action + " path: " + path);
            if (Intent.ACTION_MEDIA_MOUNTED.equals(action)) {
                // scan whenever any volume is mounted
                scan(context, MediaProvider.EXTERNAL_VOLUME);
            } else if (Intent.ACTION_MEDIA_SCANNER_SCAN_FILE.equals(action) &&
                       path != null && path.startsWith(externalStoragePath + "/")) {
                scanFile(context, path);
            }
        }
    }
}
```

onReceive方法中，会执行到scan()或者是scanFile()方法，一个是针对整个卷，一个是针对具体的文件，从传入的参数可以看出眉目，scan传入的是卷名，scanFile传入的是文件路径。通过接收广播启动扫描，然而不管是scan还是scanFile都是以startService的方式启动MediaScannerService。

``` java
private void scan(Context context, String volume) {
        Bundle args = new Bundle();
        args.putString("volume", volume);
        context.startService(
                new Intent(context, MediaScannerService.class).putExtras(args));
}

private void scanFile(Context context, String path) {
    Bundle args = new Bundle();
    args.putString("filepath", path);
    context.startService(
            new Intent(context, MediaScannerService.class).putExtras(args));
}

private void scanTranslatable(Context context) {
    final Bundle args = new Bundle();
    args.putBoolean(MediaStore.RETRANSLATE_CALL, true);
    context.startService(new Intent(context, MediaScannerService.class).putExtras(args));
}
```

##### 第三方应用发起媒体扫描的方式

#### 广播

``` java
public void scanOneFile(final String filePath) {
    Intent intent = new Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE);
    Uri uri = Uri.parse("file://" + filePath);
    intent.setData(uri);
    sendBroadcast(intent);
}
```

#### 调用API接口

``` java
MediaScannerConnection.scanFile(context, paths, null, null);
```

### MediaScannerService服务

继承自Service，实现Runnable接口，首先执行onCreate()方法：

``` java
@Override
public void onCreate() {
    PowerManager pm = (PowerManager)getSystemService(Context.POWER_SERVICE);
    mWakeLock = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, TAG);
    StorageManager storageManager = (StorageManager)getSystemService(Context.STORAGE_SERVICE);
    mExternalStoragePaths = storageManager.getVolumePaths();
    
    // Start up the thread running the service.  Note that we create a
    // separate thread because the service normally runs in the process's
    // main thread, which we don't want to block.
    Thread thr = new Thread(null, this, "MediaScannerService");
    thr.start();
}
```

这里会单独启动一个线程来处理，然后走到onStartCommand()：

  ``` java
@Override
public int onStartCommand(Intent intent, int flags, int startId) {
    while (mServiceHandler == null) {
        synchronized (this) {
            try {
                wait(100);
            } catch (InterruptedException e) {
            }
        }
    }
    
    if (intent == null) {
        Log.e(TAG, "Intent is null in onStartCommand: ",
              new NullPointerException());
        return Service.START_NOT_STICKY;
    }
    
    Message msg = mServiceHandler.obtainMessage();
    msg.arg1 = startId;
    msg.obj = intent.getExtras();
    mServiceHandler.sendMessage(msg);
    
    // Try again later if we are killed before we can finish scanning.
    return Service.START_REDELIVER_INTENT;
}
  ```

这里先等待mServiceHandler创建完成，然后通过handler message机制传递消息给handler处理，这里的mServiceHandler创建过程是在run(）方法中：

```java
@Override
public void run() {
    // reduce priority below other background threads to avoid interfering
    // with other services at boot time.
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND +
                              Process.THREAD_PRIORITY_LESS_FAVORABLE);
    Looper.prepare();
    
    mServiceLooper = Looper.myLooper();
    mServiceHandler = new ServiceHandler();
    
    Looper.loop();
}
```

由于这个是单独的线程，无法保证在发送消息的时候，handler已经创建完成，所以需要等待handler的创建
看下这个处理消息的handler，主要是调用scanFile、scan方法，两者都是会通过createMediaScanner创建MediaScanner对象，然后通过scanDirectories或者scanSingleFile来对卷或者文件进行扫描。

```java
private final class ServiceHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        Bundle arguments = (Bundle) msg.obj;
        if (arguments == null) {
            Log.e(TAG, "null intent, b/20953950");
            return;
        }
        String filePath = arguments.getString("filepath");
        
        try {
            if (filePath != null) {
                IBinder binder = arguments.getIBinder("listener");
                IMediaScannerListener listener = 
                    (binder == null ? null : IMediaScannerListener.Stub.asInterface(binder));
                Uri uri = null;
                try {
                    uri = scanFile(filePath, arguments.getString("mimetype"));
                } catch (Exception e) {
                    Log.e(TAG, "Exception scanning file", e);
                }
                if (listener != null) {
                    listener.scanCompleted(filePath, uri);
                }
            } else if (arguments.getBoolean(MediaStore.RETRANSLATE_CALL)) {
                ContentProviderClient mediaProvider = getBaseContext().getContentResolver()
                    .acquireContentProviderClient(MediaStore.AUTHORITY);
                mediaProvider.call(MediaStore.RETRANSLATE_CALL, null, null);
            } else {
                String volume = arguments.getString("volume");
                String[] directories = null;
                
                if (MediaProvider.INTERNAL_VOLUME.equals(volume)) {
                    // scan internal media storage
                    directories = new String[] {
                        Environment.getRootDirectory() + "/media",
                        Environment.getOemDirectory() + "/media",
                        Environment.getProductDirectory() + "/media",
                    };
                }
                else if (MediaProvider.EXTERNAL_VOLUME.equals(volume)) {
                    // scan external storage volumes
                    if (getSystemService(UserManager.class).isDemoUser()) {
                        directories = ArrayUtils.appendElement(String.class,
                                                               mExternalStoragePaths,
                                                               Environment.getDataPreloadsMediaDirectory().getAbsolutePath());
                    } else {
                        directories = mExternalStoragePaths;
                    }
                }
                
                if (directories != null) {
                    if (false) Log.d(TAG, "start scanning volume " + volume + ": "
                                     + Arrays.toString(directories));
                    scan(directories, volume);
                    if (false) Log.d(TAG, "done scanning volume " + volume);
                }
            }
        } catch (Exception e) {
            Log.e(TAG, "Exception in handleMessage", e);
        }
        
        stopSelf(msg.arg1);
    }
}
```

### MediaScanner
这个类很复杂，有两千行左右。
此类和MediaProvider交互频繁。在分析的时候要时刻回到MediaProvider去看看。而且该类与so包中的native函数也是互相调用。

#### 时序图

![MediaScanner sequence diagram](https://blankting.github.io/img/mediascanner_sequence_diagram.jpg)

#### 初始化

``` java
public class MediaScanner implements AutoCloseable {
    ...
    static {
        System.loadLibrary("media_jni");
        native_init();
    }
    ...
     private final MyMediaScannerClient mClient = new MyMediaScannerClient();
    ...
    public MediaScanner(Context c, String volumeName) {
        native_setup();//调用jni层的初始化
        mContext = c;
        mPackageName = c.getPackageName();
        mVolumeName = volumeName;

        ...

        mMediaProvider = mContext.getContentResolver()
                .acquireContentProviderClient(MediaStore.AUTHORITY);

      
        mAudioUri = Audio.Media.getContentUri(volumeName);
        mVideoUri = Video.Media.getContentUri(volumeName);
        mImagesUri = Images.Media.getContentUri(volumeName);
        mThumbsUri = Images.Thumbnails.getContentUri(volumeName);
        mFilesUri = Files.getContentUri(volumeName);
        mFilesUriNoNotify = mFilesUri.buildUpon().appendQueryParameter("nonotify", "1").build();
        ...
      }
}
```

#### scanDirectories

此方法由MediaScannerService调用。

``` java
public void scanDirectories(String[] directories) {
    try {
        long start = System.currentTimeMillis();
        prescan(null, true);//扫描前的预处理  
        long prescan = System.currentTimeMillis();
        if (ENABLE_BULK_INSERTS) {
            // create MediaInserter for bulk inserts
            mMediaInserter = new MediaInserter(mMediaProvider, 500);
        }
        
        for (int i = 0; i < directories.length; i++) {
            //扫描文件夹，这里有一个很重要的参数 mClient  
            // processDirectory是一个native函数 
            processDirectory(directories[i], mClient);
        }
        
        if (ENABLE_BULK_INSERTS) {
            // flush remaining inserts
            mMediaInserter.flushAll();
            mMediaInserter = null;
        }
        
        long scan = System.currentTimeMillis();
        postscan(directories);//扫描后处理
        long end = System.currentTimeMillis();
    } 
    ...
}
```

prescan大致就是创建一个FileCache，用来缓存扫描文件的一些信息，例如last_modified等。这个FileCache是从MediaProvider中已有信息构建出来的，也就是历史信息。后面根据扫描得到的新信息来对应更新历史信息。
postscan,这个函数做一些清除工作，例如以前有video生成了一些缩略图，现在video文件被干掉了，则对应的缩略图也要被干掉。
processFile注意这里传入的参数有路径和mClient，在frameworks\base\media\jni\android_media_MediaScanner.cpp中定义。mClient，这个是从MediaScannerClient派生下来的一个东西，里边保存了一个文件的一些信息。这个作为代理对象，被传入jni层，作为jni层与java层的通信媒介。

#### handleStringTag

拼接values的键值对

这个方法主要处理那些在底层解析后的数据返回到java层。

#### doScanFile

jni层调用，进入java层

``` java
public Uri doScanFile(String path, String mimeType, long lastModified, long fileSize, boolean isDirectory, boolean scanAlways, boolean noMedia) {
    ...
    // rescan for metadata if file was modified since last scan
    if (entry != null && (entry.mLastModifiedChanged || scanAlways)) {
        if (noMedia) {
            result = endFile(entry, false, false, false, false, false);
        } else {
            
            // we only extract metadata for audio and video files
            if (isaudio || isvideo) {
                processFile(path, mimeType, this);//处理视频文件，又进入jni层了
            }
            
            if (isimage) {
                processImageFile(path);//处理图片文件，又进入jni层了
            }
            ...
            result = endFile(entry, ringtones, notifications, alarms, music, podcasts);
        }
    }
    ...
    return result;
}
```

#### endFile

 数据库操作，将前面的处理结果保存。

``` java
private Uri endFile(FileEntry entry, boolean ringtones, boolean notifications,
                boolean alarms, boolean music, boolean podcasts)
                throws RemoteException {
   ...
   result = mMediaProvider.insert(tableUri, values);
   ...
   mMediaProvider.update(result, values, null, null);
}
```

### MeidaProvider

继承自ContentProvider，封装了数据库的创建，数据库的增删改查操作。

## MTP

通过MTP拷贝文件到手机，这个扫描流程是怎样的呢？

​    PC发SendObjectInfo命令给MtpServer。MtpServer需要检查存储设备剩余空间、可支持的最大文件大小。如果一切正常的话，它会通过MediaProvider的insert函数往媒体数据库中加入一条数据项。 

接着PC通过SendObject将文件内容传递给给MtpServer。而MtpServer就会创建该文件，并把数据写到文件中。 当文件数据发送完毕，MtpServer调用endSendObject。而endObject则会触发MediaScanner进行媒体文件扫描。当然，扫描完后，该文件携带的媒体信息（假如是MP3文件的话，则会把专辑信息、歌手、流派、长度等内容）加入到媒体数据库中。 
最终通过调用MtpDatabase.java中的endSendObject方法调用MediaScanner的scanMtpFile执行文件扫描。
