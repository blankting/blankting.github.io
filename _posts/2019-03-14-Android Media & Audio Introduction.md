---
layout:     post
title:      MediaPlayer&Audio Introduction
subtitle:   音频相关使用介绍
date:       2019-03-14
author:     Blank Ting
header-img: img/in-post/post-bg-android.jpg
catalog: 	 true
tags:
    - Android
    - MediaPlayer
    - AudioTrack
    - AudioRecord
    - MediaSession
---

# 音频相关介绍

## [MediaPlayer](https://developer.android.com/reference/android/media/MediaPlayer)

### MediaPlayer生命周期图

![MediaPlayer生命周期](https://blankting.github.io/img/in-post/MediaPlayerLifeCircle.gif)

### MediaPlayer使用方法

MediaPlayer支持AAC、AMR、FLAC、MP3、MIDI、OGG、PCM等格式，MediaPlayer可以通过设置元数据和播放源来音频。

1. 播放元数据

   ```java
   //直接创建，不需要设置setDataSource
   MediaPlayer mMediaPlayer；
   mMediaPlayer=MediaPlayer.create(this, R.raw.audio); 
   mMediaPlayer.start();
   ```

2. 播放普通音频文件

   ```java
   //本地音乐文件
   path=Environment.getExternalStorageDirectory()+"/xxx.wav";
   mMediaPlayer.setDataSource(path) ;
   
   //从网络加载音乐
   mMediaPlayer.setDataSource("http://..../xxx.mp3") ;
   
   //需使用异步缓冲
   mMediaPlayer.prepareAsync() ;
   ```

   ###### 几种重载的方法：

   [setDataSource(String path)](https://developer.android.com/reference/android/media/MediaPlayer.html#setDataSource(java.lang.String))

   [setDataSource(FileDescriptor fd)](https://developer.android.com/reference/android/media/MediaPlayer.html#setDataSource(java.io.FileDescriptor))

   [setDataSource(Context context,Uri uri)](https://developer.android.com/reference/android/media/MediaPlayer.html#setDataSource(android.content.Context,%20android.net.Uri))

   [setDataSource(FileDescptor fd,long offset,long length)](https://developer.android.com/reference/android/media/MediaPlayer.html#setDataSource(java.io.FileDescriptor,%20long,%20long))

   [setDataSource(MediaDataSource)](https://developer.android.com/reference/android/media/MediaPlayer.html#setDataSource(android.media.MediaDataSource))

   ###### TIPS: 尽量使用异步prepareAync()，这样不会阻塞UI线程。

3. 常用方法

   ```java
   start();//开始播放
   pause();//暂停播放
   reset()//清空MediaPlayer中的数据
   setLooping(boolean);//设置是否循环播放
   seekTo(msec)//定位到音频数据的位置，单位毫秒
   stop();//停止播放
   relase();//释放资源
   ```

## [AudioTrack](https://developer.android.com/reference/android/media/AudioTrack)

### AudioTrack简介

AudioTrack属于更偏底层的音频播放，MediaPlayerService的内部就是使用了AudioTrack。AudioTrack用于单个音频播放和管理，相比于MediaPlayer具有精炼、高效的优点,更适合实时产生播放数据的情况，如加密的音频，MediaPlayer是束手无策的，AudioTrack却可以。

AudioTrack用于播放PCM(PCM无压缩的音频格式)音乐流的回放，如果要播需放其它格式音频，需要相应的解码器，这也是AudioTrack用的比较少的原因，需要自己解码音频，通过write(byte[], int, int), write(short[], int, int)等方法推送解码数据到AudioTrack。 

### AudioTreack的2种播放模式

1. 静态模式—static
   静态的言下之意就是数据一次性交付给接收方。好处是简单高效，只需要进行一次操作就完成了数据的传递;缺点当然也很明显，对于数据量较大的音频回放，显然它是无法胜任的，因而通常只用于播放铃声、系统提醒等对内存小的操作

2. 流模式streaming
   流模式和网络上播放视频是类似的，即数据是按照一定规律不断地传递给接收方的。理论上它可用于任何音频播放的场景，不过我们一般在以下情况下采用：
   ​    音频文件过大
   ​    音频属性要求高，比如采样率高、深度大的数据
   ​    音频数据是实时产生的，这种情况就只能用流模式了

## [AudioRecord](https://developer.android.com/reference/android/media/AudioRecord)

### AudioRecord简介

AudioRecord 的主要功能是让应用能够管理音频资源，以便它们通过此类能够录制平台的声音输入硬件所收集的声音。此功能的实现就是通过 "pulling 同步"（reading读取）AudioRecord 对象的声音数据来完成的。在录音过程中，应用所需要做的就是通过后面三个类方法中的一个去及时地获取 AudioRecord 对象的录音数据。 AudioRecord 类提供的三个获取声音数据的方法分别是 read(byte[], int, int), read(short[], int, int), read(ByteBuffer, int)。无论选择使用那一个方法都必须事先设定方便用户的声音数据的存储格式。

开始录音的时候，一个 AudioRecord 需要初始化一个相关联的声音buffer，这个 buffer 主要是用来保存新的声音数据。这个 buffer 的大小，我们可以在对象构造期间去指定。它表明一个 AudioRecord 对象还没有被读取（同步）声音数据前能录多长的音(即一次可以录制的声音容量)。声音数据从音频硬件中被读出，数据大小不超过整个录音数据的大小（可以分多次读出），即每次读取初始化 buffer 容量的数据。

### AudioRecord使用方法

采集工作很简单，我们只需要构造一个AudioRecord对象，然后传入各种不同配置的参数即可。一般情况下录音实现的简单流程如下：

1. 音频源:我们可以使用麦克风作为采集音频的数据源。
2. 采样率:一秒钟对声音数据的采样次数，采样率越高，音质越好。
3. 音频通道：单声道，双声道等，
4. 音频格式:一般选用PCM格式，即原始的音频样本。
5. 缓冲区大小:音频数据写入缓冲区的总数，可以通过AudioRecord.getMinBufferSize获取最小的缓冲区。（将音频采集到缓冲区中然后再从缓冲区中读取）。

```java
public class AudioRecorder {
    private static AudioRecorder audioRecorder;
    // 音频源：音频输入-麦克风
    private final static int AUDIO_INPUT = MediaRecorder.AudioSource.MIC;
    // 采样率
    // 44100是目前的标准，但是某些设备仍然支持22050，16000，11025
    // 采样频率一般共分为22.05KHz、44.1KHz、48KHz三个等级
    private final static int AUDIO_SAMPLE_RATE = 16000;
    // 音频通道 单声道
    private final static int AUDIO_CHANNEL = AudioFormat.CHANNEL_IN_MONO;
    // 音频格式：PCM编码
    private final static int AUDIO_ENCODING = AudioFormat.ENCODING_PCM_16BIT;
    // 缓冲区大小：缓冲区字节大小
    private int bufferSizeInBytes = 0;
    // 录音对象
    private AudioRecord audioRecord;
    // 录音状态
    private Status status = Status.STATUS_NO_READY;
    // 文件名
    private String fileName;
    // 录音文件集合
    private List<String> filesName = new ArrayList<>();

    private AudioRecorder() {
    }

    //单例模式
    public static AudioRecorder getInstance() {
        if (audioRecorder == null) {
            audioRecorder = new AudioRecorder();
        }
        return audioRecorder;
    }

    /**
     * 创建录音对象
     */
    public void createAudio(String fileName, int audioSource, int sampleRateInHz, int channelConfig, int audioFormat) {
        // 获得缓冲区字节大小
        bufferSizeInBytes = AudioRecord.getMinBufferSize(sampleRateInHz,
                channelConfig, audioFormat);
        audioRecord = new AudioRecord(audioSource, sampleRateInHz, channelConfig, audioFormat, bufferSizeInBytes);
        this.fileName = fileName;
    }

    /**
     * 创建默认的录音对象
     * @param fileName 文件名
     */
    public void createDefaultAudio(String fileName) {
        mContext = ctx;
        mHandler = handler;
        // 获得缓冲区字节大小
        bufferSizeInBytes = AudioRecord.getMinBufferSize(AUDIO_SAMPLE_RATE,
                AUDIO_CHANNEL, AUDIO_ENCODING);
        audioRecord = new AudioRecord(AUDIO_INPUT, AUDIO_SAMPLE_RATE, AUDIO_CHANNEL, AUDIO_ENCODING, bufferSizeInBytes);
        this.fileName = fileName;
        status = Status.STATUS_READY;
    }

    /**
     * 开始录音
     * @param listener 音频流的监听
     */
    public void startRecord(final RecordStreamListener listener) {

        if (status == Status.STATUS_NO_READY || TextUtils.isEmpty(fileName)) {
            throw new IllegalStateException("录音尚未初始化,请检查是否禁止了录音权限~");
        }
        if (status == Status.STATUS_START) {
            throw new IllegalStateException("正在录音");
        }
        Log.d("AudioRecorder","===startRecord==="+audioRecord.getState());
        audioRecord.startRecording();

        new Thread(new Runnable() {
            @Override
            public void run() {
                writeDataTOFile(listener);
            }
        }).start();
    }

    /**
     * 停止录音
     */
    public void stopRecord() {
        Log.d("AudioRecorder","===stopRecord===");
        if (status == Status.STATUS_NO_READY || status == Status.STATUS_READY) {
            throw new IllegalStateException("录音尚未开始");
        } else {
            audioRecord.stop();
            status = Status.STATUS_STOP;
            release();
        }
    }

    /**
     * 取消录音
     */
    public void canel() {
        filesName.clear();
        fileName = null;
        if (audioRecord != null) {
            audioRecord.release();
            audioRecord = null;
        }
        status = Status.STATUS_NO_READY;
    }

    /**
     * 释放资源
     */
    public void release() {
        Log.d("AudioRecorder","===release===");
        //假如有暂停录音
        try {
            if (filesName.size() > 0) {
                List<String> filePaths = new ArrayList<>();
                for (String fileName : filesName) {
                    filePaths.add(FileUtils.getPcmFileAbsolutePath(fileName));
                }
                //清除
                filesName.clear();
                //将多个pcm文件转化为wav文件
                mergePCMFilesToWAVFile(filePaths);

            } else {
                //这里由于只要录音过filesName.size都会大于0,没录音时fileName为null
                //会报空指针 NullPointerException
                // 将单个pcm文件转化为wav文件
                //Log.d("AudioRecorder", "=====makePCMFileToWAVFile======");
                //makePCMFileToWAVFile();
            }
        } catch (IllegalStateException e) {
            throw new IllegalStateException(e.getMessage());
        }

        if (audioRecord != null) {
            audioRecord.release();
            audioRecord = null;
        }
        status = Status.STATUS_NO_READY;
    }

    /**
     * 将音频信息写入文件
     * @param listener 音频流的监听
     */
    private void writeDataTOFile(RecordStreamListener listener) {
        // new一个byte数组用来存一些字节数据，大小为缓冲区大小
        byte[] audiodata = new byte[bufferSizeInBytes];

        FileOutputStream fos = null;
        int readsize = 0;
        try {
            String currentFileName = fileName;
            if (status == Status.STATUS_PAUSE) {
                //假如是暂停录音 将文件名后面加个数字,防止重名文件内容被覆盖
                currentFileName += filesName.size();
            }
            filesName.add(currentFileName);
            File file = new File(FileUtils.getPcmFileAbsolutePath(currentFileName));
            if (file.exists()) {
                file.delete();
            }
            fos = new FileOutputStream(file);// 建立一个可存取字节的文件
        } catch (IllegalStateException e) {
            Log.e("AudioRecorder", e.getMessage());
            throw new IllegalStateException(e.getMessage());
        } catch (FileNotFoundException e) {
            Log.e("AudioRecorder", e.getMessage());
        }
        //将录音状态设置成正在录音状态
        status = Status.STATUS_START;
        while (status == Status.STATUS_START) {
            readsize = audioRecord.read(audiodata, 0, bufferSizeInBytes);
            if (AudioRecord.ERROR_INVALID_OPERATION != readsize && fos != null) {
                try {
                    fos.write(audiodata);
                    if (listener != null) {
                        //用于拓展业务
                        listener.recordOfByte(audiodata, 0, audiodata.length);
                    }
                } catch (IOException e) {
                    Log.e("AudioRecorder", e.getMessage());
                }
            }
        }
        try {
            if (fos != null) {
                fos.close();// 关闭写入流
            }
        } catch (IOException e) {
            Log.e("AudioRecorder", e.getMessage());
        }
    }
}
```

## [MediaSession](https://developer.android.com/reference/android/media/session/MediaSession)

### MediaSession简介

MediaSession 框架是 Google 推出专门解决媒体播放时界面和服务通讯问题。这个框架可以让我们不再使用广播来控制播放器，而且也能适配耳机，蓝牙等一些其它设备，实现线控的功能

首先Media是媒体的意思，也就是说这个框架用于音视频媒体；而Session呢，翻译成中文就是会话的意思。一个会话，肯定是涉及两方或以上；在MediaSession框架中，有受控端（一个）和控制端（可以有多个）。接下来为了保证受控端和控制端不串号（想象一个遥控器可以遥控同一型号的多台电视），就有了SessionToken的概念，相当于我们在连接蓝牙设备时的配对码，这样就保证了不串号。

### MediaSession使用方法

```java
/**
 * 初始化并激活 MediaSession
 */
 private void setupMediaSession() {
 //       第二个参数 tag: 这个是用于调试用的,随便填写即可
     mMediaSession = new MediaSessionCompat(context, TAG);
     //指明支持的按键信息类型
     mMediaSession.setFlags(
          MediaSessionCompat.FLAG_HANDLES_MEDIA_BUTTONS |
                  MediaSessionCompat.FLAG_HANDLES_TRANSPORT_CONTROLS
     );
     mMediaSession.setCallback(callback);
     mMediaSession.setActive(true);
 }
```

```java
/**
 *接收消息类型
 **/
private static final long MEDIA_SESSION_ACTIONS =
            PlaybackStateCompat.ACTION_PLAY
                    | PlaybackStateCompat.ACTION_PAUSE
                    | PlaybackStateCompat.ACTION_PLAY_PAUSE
                    | PlaybackStateCompat.ACTION_SKIP_TO_NEXT
                    | PlaybackStateCompat.ACTION_SKIP_TO_PREVIOUS
                    | PlaybackStateCompat.ACTION_STOP
                    | PlaybackStateCompat.ACTION_SEEK_TO;
```

``` java
/**
 * 耳机多媒体按钮监听 MediaSessionCompat.Callback
 */
private MediaSessionCompat.Callback callback = new MediaSessionCompat.Callback() {
//        接收到监听事件，可以有选择的进行重写相关方法
    @Override
    public void onPlay() {
        super.onPlay();
    }

    @Override
    public void onPause() {
        super.onPause();
    }

    @Override
    public void onSkipToNext() {
        super.onSkipToNext();
    }

    @Override
    public void onSkipToPrevious() {
        super.onSkipToPrevious();
    }

    @Override
    public void onStop() {
        super.onStop();
    }

    @Override
    public void onSeekTo(long pos) {
        super.onSeekTo(pos);
    }
}；
```

``` java
/**
 * 更新播放状态，播放/暂停/拖动进度条时调用
 */
public void updatePlaybackState() {
    int state = isPlaying() ？ PlaybackStateCompat.STATE_PLAYING :                   				PlaybackStateCompat.STATE_PAUSED;
    mMediaSession.setPlaybackState(
            new PlaybackStateCompat.Builder()
                    .setActions(MEDIA_SESSION_ACTIONS)
                    .setState(state, getCurrentPosition(), 1)
                    .build());
}
```

``` java
/**
 * 更新正在播放的音乐信息，切换歌曲时调用
 */
public void updateMetaData(String path) {
    if (!StringUtils.isReal(path)) {
        mMediaSession.setMetadata(null);
        return;
    }

    SongInfo info = mediaManager.getSongInfo(context, path);
    MediaMetadataCompat.Builder metaData = new MediaMetadataCompat.Builder()
            .putString(MediaMetadataCompat.METADATA_KEY_TITLE, info.getTitle())
            .putString(MediaMetadataCompat.METADATA_KEY_ARTIST, info.getArtist())
            .putString(MediaMetadataCompat.METADATA_KEY_ALBUM, info.getAlbum())
            .putString(MediaMetadataCompat.METADATA_KEY_ALBUM_ARTIST, info.getArtist())
            .putLong(MediaMetadataCompat.METADATA_KEY_DURATION, info.getDuration())
            .putBitmap(MediaMetadataCompat.METADATA_KEY_ALBUM_ART, getCoverBitmap(info));
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        metaData.putLong(MediaMetadataCompat.METADATA_KEY_NUM_TRACKS, getCount());
    }
    mMediaSession.setMetadata(metaData.build());
}
```

