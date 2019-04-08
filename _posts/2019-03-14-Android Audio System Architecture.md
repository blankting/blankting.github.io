---
layout:     post
title:      Android Audio System Architecture introduction
subtitle:   Android Audio深入理解
date:       2019-03-14
author:     Blank Ting
header-img: img/in-post/post-bg-android.jpg
catalog: 	 true
tags:
    - Android
    - Framework
    - MultiMedia
    - Audio
---

# Android Audio System Architecture introduction

## 总览

### 两张图

![Audio System Architecture](https://blankting.github.io/img/in-post/AudioSystemArchtitecture.jpg)

![Audio code structure](https://blankting.github.io/img/in-post/CodeStructure.jpeg)



## Application层

这是整个音频体系的最上层，因而并不是Android系统实现的重点。比如厂商根据特定需求自己写的一个音乐播放器，游戏中使用到声音，或者调节音频的一类软件等等。

音频相关的应用软件有： 音乐播放器，FM Radio，电话，声音设置，视频播放器等等。

## Application Framework层

相信大家可以马上想到MediaPlayer和MediaRecorder，因为这是我们在开发音频相关产品时使用最广泛的两个类。实际上，Android也提供了另两个相似功能的类，即AudioTrack和AudioRecorder，MediaPlayerService内部的实现就是通过它们来完成的,只不过MediaPlayer/MediaRecorder提供了更强大的控制功能，相比前者也更易于使用。我们后面还会有详细介绍。

除此以外，Android系统还为我们控制音频系统提供了AudioManager、AudioService及AudioSystem类。这些都是framework为便利上层应用开发所设计的。

该层代码位置frameworks/base/media/java/android/media。其中关键类的作用如下：

MediaPlayer和MediaRecorder，AudioTrack和AudioRecorder，提供声音播放和录制接口，但是MediaPlayer/MediaRecorder功能更强大，也更易于使用。
AudioManager、AudioService及AudioSystem等类提供声音控制、通道选择、音效设置等功能。

## Jni

Audio的JNI代码在framewoks/base/core/jni目录下面，会和其他一些系统文件生成libandroid_runtime.so供上层调用。

## Native Framework层

我们知道，framework层的很多类，实际上只是应用程序使用Android库文件的“中介”而已。因为上层应用采用java语言编写，它们需要最直接的java接口的支持，这就是framework层存在的意义之一。而作为“中介”，它们并不会真正去实现具体的功能，或者只实现其中的一部分功能，而把主要重心放在库中来完成。比如上面的AudioTrack、AudioRecorder、MediaPlayer和MediaRecorder等等在库中都能找到相对应的类。

这一部分代码集中放置在工程的frameworks/av/media/libmedia中，多数是C++语言编写的。

除了上面的类库实现外，音频系统还需要一个“核心中控”，或者用Android中通用的实现来讲，需要一个系统服务(比如ServiceManager、LocationManagerService、ActivityManagerService等等)，这就是AudioFlinger和AudioPolicyService。它们的代码放置在frameworks/av/services/audioflinger，生成的最主要的库叫做libaudioflinger。

音频体系中另一个重要的系统服务是MediaPlayerService，它的位置在frameworks/av/media/libmediaplayerservice。

因为涉及到的库和相关类是非常多的，建议大家在理解的时候分为两条线索。

其一，以库为线索。比如AudioPolicyService和AudioFlinger都是在libaudioflinger库中;而AudioTrack、AudioRecorder等一系列实现则在libmedia库中。

其二，以进程为线索。库并不代表一个进程，进程则依赖于库来运行。虽然有的类是在同一个库中实现的，但并不代表它们会在同一个进程中被调用。比如AudioFlinger和AudioPolicyService都驻留于名为mediaserver的系统进程中;而AudioTrack/AudioRecorder和MediaPlayer/MediaRecorder一样实际上只是应用进程的一部分，它们通过binder服务来与其它系统进程通信。

在分析源码的过程中，一定要紧抓这两条线索，才不至于觉得混乱。

### 客户端： 
       AudioTrack、AudioRecorder、MediaPlayer、MediaRecorder、AudioSystem对应Java层的相关类，代码放置在frameworks/av/media/libmedia中， C++语言编写，编译后成为libmedia库。

#### AudioTrack Native API 四种数据传输模式：

| Transfer Mode     | Description                                                  |
| ----------------- | ------------------------------------------------------------ |
| TRANSFER_CALLBACK | 在 AudioTrackThread 线程中通过 audioCallback 回调函数主动从应用进程那里索取数据，ToneGenerator 采用这种模式 |
| TRANSFER_OBTAIN   | 应用进程需要调用 obtainBuffer()/releaseBuffer() 填充数据，目前我还没有见到实际的使用场景 |
| TRANSFER_SYNC     | 应用进程需要持续调用 write() 写数据到 FIFO，写数据时有可能遭遇阻塞（等待 AudioFlinger::PlaybackThread 消费之前的数据），基本适用所有的音频场景；对应于 AudioTrack Java API 的 MODE_STREAM 模式 |
| TRANSFER_SHARED   | 应用进程将回放数据一次性付给 AudioTrack，适用于数据量小、时延要求高的场景；对应于 AudioTrack Java API 的 MODE_STATIC 模式 |

#### AudioTrack Native API 音频流类型：

| Stream Type               | Description                      |
| ------------------------- | -------------------------------- |
| AUDIO_STREAM_VOICE_CALL   | 电话语音                         |
| AUDIO_STREAM_SYSTEM       | 系统声音                         |
| AUDIO_STREAM_RING         | 铃声声音，如来电铃声、闹钟铃声等 |
| AUDIO_STREAM_MUSIC        | 音乐声音                         |
| AUDIO_STREAM_ALARM        | 警告音                           |
| AUDIO_STREAM_NOTIFICATION | 通知音                           |
| AUDIO_STREAM_DTMF         | DTMF 音（拨号盘按键音）          |

#### AudioTrack Native API 输出标识：

| AUDIO_OUTPUT_FLAG                  | Description                                                  |
| ---------------------------------- | ------------------------------------------------------------ |
| AUDIO_OUTPUT_FLAG_DIRECT           | 表示音频流直接输出到音频设备，不需要软件混音，一般用于 HDMI 设备声音输出 |
| AUDIO_OUTPUT_FLAG_PRIMARY          | 表示音频流需要输出到主输出设备，一般用于铃声类声音           |
| AUDIO_OUTPUT_FLAG_FAST             | 表示音频流需要快速输出到音频设备，一般用于按键音、游戏背景音等对时延要求高的场景 |
| AUDIO_OUTPUT_FLAG_DEEP_BUFFER      | 表示音频流输出可以接受较大的时延，一般用于音乐、视频播放等对时延要求不高的场景 |
| AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD | 表示音频流没有经过软件解码，需要输出到硬件解码器，由硬件解码器进行解码 |

Android 为什么要定义这么多的流类型？这与 Android 的音频管理策略有关，例如：

- 音频流的音量管理，调节一个类型的音频流音量，不会影响到其他类型的音频流
- 根据流类型选择合适的输出设备；比如插着有线耳机期间，音乐声（STREAM_MUSIC）只会输出到有线耳机，而铃声（STREAM_RING）会同时输出到有线耳机和外放

### 服务端： 
       AudioFlinger和AudioPolicyService是核心代码，它们的代码在frameworks/av/services/audioflinger，编译后成为libaudioflinger库，运行在AudioServer系统进程（从 Android 7.0 开始，之前版本由 mediaserver 加载）。

#### AudioFlinger

音频系统策略的执行者，负责音频流设备的管理及音频流数据的处理传输，AudioTrack称为音轨，负责音频数据的传送，把数据交给AudioFlinger，AudioFlinger可以称之为混音器，同时可以进行音效的处理，然后把处理后的数据交给Audio Hal层，最终由相应的硬件播放，启到承上启下的作用。所以 AudioFlinger 也被认为是 Android 音频系统的引擎。

AudioFlinger dump: adb shell dumpsys media.audio_flinger

AudioFlinger 对外提供的主要的服务接口如下：

| Interface       | Description                                                  |
| --------------- | ------------------------------------------------------------ |
| sampleRate      | 获取硬件设备的采样率                                         |
| format          | 获取硬件设备的音频格式                                       |
| frameCount      | 获取硬件设备的周期帧数                                       |
| latency         | 获取硬件设备的传输延迟                                       |
| setMasterVolume | 调节主输出设备的音量                                         |
| setMasterMute   | 静音主输出设备                                               |
| setStreamVolume | 调节指定类型的音频流的音量，这种调节不影响其他类型的音频流的音量 |
| setStreamMute   | 静音指定类型的音频流                                         |
| setVoiceVolume  | 调节通话音量                                                 |
| setMicMute      | 静音麦克风输入                                               |
| setMode         | 切换音频模式：音频模式有 4 种，分别是 Normal、Ringtone、Call、Communicatoin |
| setParameters   | 设置音频参数：往下调用 HAL 层相应接口，常用于切换音频通道    |
| getParameters   | 获取音频参数：往下调用 HAL 层相应接口                        |
| openOutput      | 打开输出流：打开输出流设备，并创建 PlaybackThread 对象       |
| closeOutput     | 关闭输出流：移除并销毁 PlaybackThread 上面挂着的所有的 Track，退出 PlaybackThread，关闭输出流设备 |
| openInput       | 打开输入流：打开输入流设备，并创建 RecordThread 对象         |
| closeInput      | 关闭输入流：退出 RecordThread，关闭输入流设备                |
| createTrack     | 新建输出流管理对象： 找到对应的 PlaybackThread，创建输出流管理对象 Track，然后创建并返回该 Track 的代理对象 TrackHandle |
| openRecord      | 新建输入流管理对象：找到 RecordThread，创建输入流管理对象 RecordTrack，然后创建并返回该 RecordTrack 的代理对象 RecordHandle |

可以归纳出 AudioFlinger 响应的服务请求主要有：

- 获取硬件设备的配置信息
- 音量调节
- 静音操作
- 音频模式切换
- 音频参数设置
- 输入输出流设备管理
- 音频流管理

#### AudioFlinger代码分析

AudioFlinger初始化

``` c++
//AudioFlinger.cpp (frameworks\av\services\audioflinger)
AudioFlinger::AudioFlinger()
    : BnAudioFlinger(),
      mMediaLogNotifier(new AudioFlinger::MediaLogNotifier()),
      mPrimaryHardwareDev(NULL),
      mAudioHwDevs(NULL),
      mHardwareStatus(AUDIO_HW_IDLE),
      mMasterVolume(1.0f),
      mMasterMute(false),
      // mNextUniqueId(AUDIO_UNIQUE_ID_USE_MAX),
      mMode(AUDIO_MODE_INVALID),
      mBtNrecIsOff(false),
      mIsLowRamDevice(true),
      mIsDeviceTypeKnown(false),
      mTotalMemory(0),
      mClientSharedHeapSize(kMinimumClientSharedHeapSizeBytes),
      mGlobalEffectEnableTime(0),
      mSystemReady(false)
{
    ...
    mDevicesFactoryHal = DevicesFactoryHalInterface::create();
    mEffectsFactoryHal = EffectsFactoryHalInterface::create();
    ...
}
```

DevicesFactory创建并打开设备

``` c++
//DevicesFactoryHalInterface.cpp (frameworks\av\media\libaudiohal)
namespace android {
// static
sp<DevicesFactoryHalInterface> DevicesFactoryHalInterface::create() {
    if (hardware::audio::V4_0::IDevicesFactory::getService() != nullptr) {
        return new V4_0::DevicesFactoryHalHybrid();
    }
    if (hardware::audio::V2_0::IDevicesFactory::getService() != nullptr) {
        return new DevicesFactoryHalHybrid();
    }
    return nullptr;
}
}
```

DevicesFactoryHalHybrid.cpp中分别创建了不同的实现对象，通过 USE_LEGACY_LOCAL_AUDIO_HAL识别，如果是本地模式，DevicesFactoryHalLocal.cpp中使用8.0以前的HAL加载方式,如果是HIDL的Treble模式，将采用8.0的新架构。

``` c++
//DevicesFactoryHalHybrid.cpp (frameworks\av\media\libaudiohal\2.0)/(frameworks\av\media\libaudiohal\4.0)
status_t DevicesFactoryHalHybrid::openDevice(const char *name, sp<DeviceHalInterface> *device) {
    if (mHidlFactory != 0 && strcmp(AUDIO_HARDWARE_MODULE_ID_A2DP, name) != 0 &&
        strcmp(AUDIO_HARDWARE_MODULE_ID_HEARING_AID, name) != 0) {
        return mHidlFactory->openDevice(name, device);
    }
    return mLocalFactory->openDevice(name, device);
}
```

HAL方式

``` c++
static status_t load_audio_interface(const char *if_name, audio_hw_device_t **dev)
{
    const hw_module_t *mod;
    int rc;

    rc = hw_get_module_by_class(AUDIO_HARDWARE_MODULE_ID, if_name, &mod);
    if (rc) {
        ALOGE("%s couldn't load audio hw module %s.%s (%s)", __func__,
                AUDIO_HARDWARE_MODULE_ID, if_name, strerror(-rc));
        goto out;
    }
    rc = audio_hw_device_open(mod, dev);
    if (rc) {
        ALOGE("%s couldn't open audio hw device in %s.%s (%s)", __func__,
                AUDIO_HARDWARE_MODULE_ID, if_name, strerror(-rc));
        goto out;
    }
    if ((*dev)->common.version < AUDIO_DEVICE_API_VERSION_MIN) {
        ALOGE("%s wrong audio hw device version %04x", __func__, (*dev)->common.version);
        rc = BAD_VALUE;
        audio_hw_device_close(*dev);
        goto out;
    }
    return OK;

out:
    *dev = NULL;
    return rc;
}

status_t DevicesFactoryHalLocal::openDevice(const char *name, sp<DeviceHalInterface> *device) {
    audio_hw_device_t *dev;
    status_t rc = load_audio_interface(name, &dev);
    if (rc == OK) {
        *device = new DeviceHalLocal(dev);
    }
    return rc;
}
```

HIDL方式

``` c++
DevicesFactoryHalHidl::DevicesFactoryHalHidl() {
    mDevicesFactory = IDevicesFactory::getService();
    if (mDevicesFactory != 0) {
        // It is assumed that DevicesFactory is owned by AudioFlinger
        // and thus have the same lifespan.
        mDevicesFactory->linkToDeath(HalDeathHandler::getInstance(), 0 /*cookie*/);
    } else {
        ALOGE("Failed to obtain IDevicesFactory service, terminating process.");
        exit(1);
    }
    // The MSD factory is optional
    mDevicesFactoryMsd = IDevicesFactory::getService(AUDIO_HAL_SERVICE_NAME_MSD);
    // TODO: Register death handler, and add 'restart' directive to audioserver.rc
}

DevicesFactoryHalHidl::~DevicesFactoryHalHidl() {
}

// static
status_t DevicesFactoryHalHidl::nameFromHal(const char *name, IDevicesFactory::Device *device) {
    if (strcmp(name, AUDIO_HARDWARE_MODULE_ID_PRIMARY) == 0) {
        *device = IDevicesFactory::Device::PRIMARY;
        return OK;
    } else if(strcmp(name, AUDIO_HARDWARE_MODULE_ID_A2DP) == 0) {
        *device = IDevicesFactory::Device::A2DP;
        return OK;
    } else if(strcmp(name, AUDIO_HARDWARE_MODULE_ID_USB) == 0) {
        *device = IDevicesFactory::Device::USB;
        return OK;
    } else if(strcmp(name, AUDIO_HARDWARE_MODULE_ID_REMOTE_SUBMIX) == 0) {
        *device = IDevicesFactory::Device::R_SUBMIX;
        return OK;
    } else if(strcmp(name, AUDIO_HARDWARE_MODULE_ID_STUB) == 0) {
        *device = IDevicesFactory::Device::STUB;
        return OK;
    }
    ALOGE("Invalid device name %s", name);
    return BAD_VALUE;
}

status_t DevicesFactoryHalHidl::openDevice(const char *name, sp<DeviceHalInterface> *device) {
    if (mDevicesFactory == 0) return NO_INIT;
    IDevicesFactory::Device hidlDevice;
    status_t status = nameFromHal(name, &hidlDevice);
    if (status != OK) return status;
    Result retval = Result::NOT_INITIALIZED;
    Return<void> ret = mDevicesFactory->openDevice(
            hidlDevice,
            [&](Result r, const sp<IDevice>& result) {
                retval = r;
                if (retval == Result::OK) {
                    *device = new DeviceHalHidl(result);
                }
            });
    if (ret.isOk()) {
        if (retval == Result::OK) return OK;
        else if (retval == Result::INVALID_ARGUMENTS) return BAD_VALUE;
        else return NO_INIT;
    }
    return FAILED_TRANSACTION;
}
```

至此，AudioFlinger就能配合AudioPolicyService创建Track了。

``` c++
sp<IAudioTrack> AudioFlinger::createTrack(const CreateTrackInput& input,
                                          CreateTrackOutput& output,
                                          status_t *status)
{
    sp<PlaybackThread::Track> track;
    sp<TrackHandle> trackHandle;
    sp<Client> client;
    status_t lStatus;
    audio_stream_type_t streamType;
    audio_port_handle_t portId = AUDIO_PORT_HANDLE_NONE;

    bool updatePid = (input.clientInfo.clientPid == -1);
    const uid_t callingUid = IPCThreadState::self()->getCallingUid();
    uid_t clientUid = input.clientInfo.clientUid;
    if (!isTrustedCallingUid(callingUid)) {
        ALOGW_IF(clientUid != callingUid,
                "%s uid %d tried to pass itself off as %d",
                __FUNCTION__, callingUid, clientUid);
        clientUid = callingUid;
        updatePid = true;
    }
    pid_t clientPid = input.clientInfo.clientPid;
    if (updatePid) {
        const pid_t callingPid = IPCThreadState::self()->getCallingPid();
        ALOGW_IF(clientPid != -1 && clientPid != callingPid,
                 "%s uid %d pid %d tried to pass itself off as pid %d",
                 __func__, callingUid, callingPid, clientPid);
        clientPid = callingPid;
    }

    audio_session_t sessionId = input.sessionId;
    if (sessionId == AUDIO_SESSION_ALLOCATE) {
        sessionId = (audio_session_t) newAudioUniqueId(AUDIO_UNIQUE_ID_USE_SESSION);
    } else if (audio_unique_id_get_use(sessionId) != AUDIO_UNIQUE_ID_USE_SESSION) {
        lStatus = BAD_VALUE;
        goto Exit;
    }

    output.sessionId = sessionId;
    output.outputId = AUDIO_IO_HANDLE_NONE;
    output.selectedDeviceId = input.selectedDeviceId;

    lStatus = AudioSystem::getOutputForAttr(&input.attr, &output.outputId, sessionId, &streamType,
                                            clientPid, clientUid, &input.config, input.flags,
                                            &output.selectedDeviceId, &portId);

    if (lStatus != NO_ERROR || output.outputId == AUDIO_IO_HANDLE_NONE) {
        ALOGE("createTrack() getOutputForAttr() return error %d or invalid output handle", lStatus);
        goto Exit;
    }
    // client AudioTrack::set already implements AUDIO_STREAM_DEFAULT => AUDIO_STREAM_MUSIC,
    // but if someone uses binder directly they could bypass that and cause us to crash
    if (uint32_t(streamType) >= AUDIO_STREAM_CNT) {
        ALOGE("createTrack() invalid stream type %d", streamType);
        lStatus = BAD_VALUE;
        goto Exit;
    }

    // further channel mask checks are performed by createTrack_l() depending on the thread type
    if (!audio_is_output_channel(input.config.channel_mask)) {
        ALOGE("createTrack() invalid channel mask %#x", input.config.channel_mask);
        lStatus = BAD_VALUE;
        goto Exit;
    }

    // further format checks are performed by createTrack_l() depending on the thread type
    if (!audio_is_valid_format(input.config.format)) {
        ALOGE("createTrack() invalid format %#x", input.config.format);
        lStatus = BAD_VALUE;
        goto Exit;
    }

    {
        Mutex::Autolock _l(mLock);
        PlaybackThread *thread = checkPlaybackThread_l(output.outputId);
        if (thread == NULL) {
            ALOGE("no playback thread found for output handle %d", output.outputId);
            lStatus = BAD_VALUE;
            goto Exit;
        }

        client = registerPid(clientPid);

        PlaybackThread *effectThread = NULL;
        // check if an effect chain with the same session ID is present on another
        // output thread and move it here.
        for (size_t i = 0; i < mPlaybackThreads.size(); i++) {
            sp<PlaybackThread> t = mPlaybackThreads.valueAt(i);
            if (mPlaybackThreads.keyAt(i) != output.outputId) {
                uint32_t sessions = t->hasAudioSession(sessionId);
                if (sessions & ThreadBase::EFFECT_SESSION) {
                    effectThread = t.get();
                    break;
                }
            }
        }
        ALOGV("createTrack() sessionId: %d", sessionId);

        output.sampleRate = input.config.sample_rate;
        output.frameCount = input.frameCount;
        output.notificationFrameCount = input.notificationFrameCount;
        output.flags = input.flags;

        track = thread->createTrack_l(client, streamType, input.attr, &output.sampleRate,
                                      input.config.format, input.config.channel_mask,
                                      &output.frameCount, &output.notificationFrameCount,
                                      input.notificationsPerBuffer, input.speed,
                                      input.sharedBuffer, sessionId, &output.flags,
                                      input.clientInfo.clientTid, clientUid, &lStatus, portId);
        LOG_ALWAYS_FATAL_IF((lStatus == NO_ERROR) && (track == 0));
        // we don't abort yet if lStatus != NO_ERROR; there is still work to be done regardless

        output.afFrameCount = thread->frameCount();
        output.afSampleRate = thread->sampleRate();
        output.afLatencyMs = thread->latency();

        // move effect chain to this output thread if an effect on same session was waiting
        // for a track to be created
        if (lStatus == NO_ERROR && effectThread != NULL) {
            // no risk of deadlock because AudioFlinger::mLock is held
            Mutex::Autolock _dl(thread->mLock);
            Mutex::Autolock _sl(effectThread->mLock);
            moveEffectChain_l(sessionId, effectThread, thread, true);
        }

        // Look for sync events awaiting for a session to be used.
        for (size_t i = 0; i < mPendingSyncEvents.size(); i++) {
            if (mPendingSyncEvents[i]->triggerSession() == sessionId) {
                if (thread->isValidSyncEvent(mPendingSyncEvents[i])) {
                    if (lStatus == NO_ERROR) {
                        (void) track->setSyncEvent(mPendingSyncEvents[i]);
                    } else {
                        mPendingSyncEvents[i]->cancel();
                    }
                    mPendingSyncEvents.removeAt(i);
                    i--;
                }
            }
        }

        setAudioHwSyncForSession_l(thread, sessionId);
    }

    if (lStatus != NO_ERROR) {
        // remove local strong reference to Client before deleting the Track so that the
        // Client destructor is called by the TrackBase destructor with mClientLock held
        // Don't hold mClientLock when releasing the reference on the track as the
        // destructor will acquire it.
        {
            Mutex::Autolock _cl(mClientLock);
            client.clear();
        }
        track.clear();
        goto Exit;
    }

    // return handle to client
    trackHandle = new TrackHandle(track);

Exit:
    if (lStatus != NO_ERROR && output.outputId != AUDIO_IO_HANDLE_NONE) {
        AudioSystem::releaseOutput(output.outputId, streamType, sessionId);
    }
    *status = lStatus;
    return trackHandle;
}
```

AudioTrack创建成功以后，开始调用start()函数，start()函数将由TrackHandle代理并调用Track的start()函数

``` c++
//Tracks.cpp (frameworks\av\services\audioflinger)
status_t AudioFlinger::PlaybackThread::Track::start(AudioSystem::sync_event_t event __unused,
                                                    audio_session_t triggerSession __unused)
{
    status_t status = NO_ERROR;
    MTK_ALOGV("start(%d), calling pid %d session %d",
            mName, IPCThreadState::self()->getCallingPid(), mSessionId);

    sp<ThreadBase> thread = mThread.promote();
    if (thread != 0) {
        if (isOffloaded()) {
            Mutex::Autolock _laf(thread->mAudioFlinger->mLock);
            Mutex::Autolock _lth(thread->mLock);
            sp<EffectChain> ec = thread->getEffectChain_l(mSessionId);
            if (thread->mAudioFlinger->isNonOffloadableGlobalEffectEnabled_l() ||
                    (ec != 0 && ec->isNonOffloadableEnabled())) {
                invalidate();
                return PERMISSION_DENIED;
            }
        }
        Mutex::Autolock _lth(thread->mLock);
        track_state state = mState;
        // here the track could be either new, or restarted
        // in both cases "unstop" the track

        // initial state-stopping. next state-pausing.
        // What if resume is called ?

        if (state == PAUSED || state == PAUSING) {
            if (mResumeToStopping) {
                // happened we need to resume to STOPPING_1
                mState = TrackBase::STOPPING_1;
                MTK_ALOGV("PAUSED => STOPPING_1 (%d) on thread %p", mName, this);
            } else {
                mState = TrackBase::RESUMING;
                MTK_ALOGV("PAUSED => RESUMING (%d) on thread %p", mName, this);
            }
#if defined(MTK_AUDIO_FIX_DEFAULT_DEFECT) // ALPS03762573 : music seek
        } else if ((state == FLUSHED || state == IDLE) &&
                    mStreamType == AUDIO_STREAM_MUSIC) {
            mState = TrackBase::RESUMING;
#endif
        } else {
            mState = TrackBase::ACTIVE;
            MTK_ALOGV("? => ACTIVE (%d) on thread %p", mName, this);
        }

        // states to reset position info for non-offloaded/direct tracks
        if (!isOffloaded() && !isDirect()
                && (state == IDLE || state == STOPPED || state == FLUSHED)) {
            mFrameMap.reset();
        }
        PlaybackThread *playbackThread = (PlaybackThread *)thread.get();
        if (isFastTrack()) {
            // refresh fast track underruns on start because that field is never cleared
            // by the fast mixer; furthermore, the same track can be recycled, i.e. start
            // after stop.
            mObservedUnderruns = playbackThread->getFastTrackUnderruns(mFastIndex);
        }
        status = playbackThread->addTrack_l(this);
        if (status == INVALID_OPERATION || status == PERMISSION_DENIED) {
            triggerEvents(AudioSystem::SYNC_EVENT_PRESENTATION_COMPLETE);
            //  restore previous state if start was rejected by policy manager
            if (status == PERMISSION_DENIED) {
                mState = state;
            }
        }

        if (status == NO_ERROR || status == ALREADY_EXISTS) {
            // for streaming tracks, remove the buffer read stop limit.
            mAudioTrackServerProxy->start();
        }

        // track was already in the active list, not a problem
        if (status == ALREADY_EXISTS) {
            status = NO_ERROR;
        } else {
            // Acknowledge any pending flush(), so that subsequent new data isn't discarded.
            // It is usually unsafe to access the server proxy from a binder thread.
            // But in this case we know the mixer thread (whether normal mixer or fast mixer)
            // isn't looking at this track yet:  we still hold the normal mixer thread lock,
            // and for fast tracks the track is not yet in the fast mixer thread's active set.
            // For static tracks, this is used to acknowledge change in position or loop.
            ServerProxy::Buffer buffer;
            buffer.mFrameCount = 1;
            (void) mAudioTrackServerProxy->obtainBuffer(&buffer, true /*ackFlush*/);
        }
    } else {
        status = BAD_VALUE;
    }
    return status;
}
```

将创建好的Track加入到mActiveTracks中

``` c++
//Tracks.cpp (frameworks\av\services\audioflinger)
status_t AudioFlinger::PlaybackThread::addTrack_l(const sp<Track>& track)
{
    status_t status = ALREADY_EXISTS;

    if (mActiveTracks.indexOf(track) < 0) {
        // the track is newly added, make sure it fills up all its
        // buffers before playing. This is to ensure the client will
        // effectively get the latency it requested.
        if (track->isExternalTrack()) {
            TrackBase::track_state state = track->mState;
            mLock.unlock();
#if !defined(MTK_HIFIAUDIO_SUPPORT)
            status = AudioSystem::startOutput(mId, track->streamType(),
                                              track->sessionId());
#else
            status = AudioSystem::startOutputSamplerate(mId, track->streamType(),
                                              track->sessionId(), track->mSampleRate);
#endif
            mLock.lock();
            // abort track was stopped/paused while we released the lock
            if (state != track->mState) {
                if (status == NO_ERROR) {
                    mLock.unlock();
#if !defined(MTK_HIFIAUDIO_SUPPORT)
                    AudioSystem::stopOutput(mId, track->streamType(),
                                            track->sessionId());
#else
                    AudioSystem::stopOutputSamplerate(mId, track->streamType(),
                                            track->sessionId(), track->mSampleRate);
#endif
                    mLock.lock();
                }
                return INVALID_OPERATION;
            }
            // abort if start is rejected by audio policy manager
            if (status != NO_ERROR) {
                return PERMISSION_DENIED;
            }
#ifdef ADD_BATTERY_DATA
            // to track the speaker usage
            addBatteryData(IMediaPlayerService::kBatteryDataAudioFlingerStart);
#endif
        }

        // set retry count for buffer fill
        if (track->isOffloaded()) {
            if (track->isStopping_1()) {
                track->mRetryCount = kMaxTrackStopRetriesOffload;
            } else {
                track->mRetryCount = kMaxTrackStartupRetriesOffload;
            }
            track->mFillingUpStatus = mStandby ? Track::FS_FILLING : Track::FS_FILLED;
        } else {
            track->mRetryCount = kMaxTrackStartupRetries;
            track->mFillingUpStatus =
                    track->sharedBuffer() != 0 ? Track::FS_FILLED : Track::FS_FILLING;
        }

        track->mResetDone = false;
        track->mPresentationCompleteFrames = 0;
        mActiveTracks.add(track);
        sp<EffectChain> chain = getEffectChain_l(track->sessionId());
        if (chain != 0) {
            ALOGV("addTrack_l() starting track on chain %p for session %d", chain.get(),
                    track->sessionId());
            chain->incActiveTrackCnt();
#if defined(MTK_AUDIO_FIX_DEFAULT_DEFECT) // ALPS03762573 : music seek
            android_atomic_release_store(false, &mFlushShareBuffer[track->name()]);
#endif
        }

        status = NO_ERROR;
    }

    onAddNewTrack_l();
    return status;
}
```

剩下的就是混音处理以及loop和刷新了。

总的来说，就是AudioFlinger通过AudioPolicyService创建混音线程，混音线程将Track中的数据取出，进入环形缓冲区处理，最终输出到音频硬件设备；

#### AudioPolicyService

音频策略的制定者，负责音频设备切换的策略抉择、音量调节策略等，AudioPolicyService的很大一部分管理工作都是在AudioPolicyManager中完成的。包括音量管理，音频策略（strategy）管理，输入输出设备管理。

AudioPolicyService在Android Audio系统中主要完成以下几个任务： 

1. 管理输入输出设备，包括设备的连接、断开状态，设备的选择和切换等 
2. 管理系统的音频策略，比如通话时播放音乐、或者播放音乐时来电话的一系列处理 
3. 管理系统的音量 
4. 上层的一些音频参数也可以通过AudioPolicyService设置到底层去

#### AudioPolicyManager

在创建过程中会通过加载audio_policy.conf配置文件来加载音频设备,Android为每种音频接口定义了对应的硬件抽象层。每种音频接口定义了不同的输入输出，一个接口可以具有多个输入或者输出，每个输入输出可以支持不同的设备，通过读取audio_policy.conf文件可以获取系统支持的音频接口参数，在AudioPolicyManager中会优先加载/vendor/etc/audio_policy.conf配置文件, 如果该配置文件不存在, 则加载/system/etc/audio_policy.conf配置文件。AudioPolicyManager加载完所有音频接口后,就知道了系统支持的所有音频接口参数,可以为音频输出提供决策。

#### AudioPolicyService+AudioPolicyManager代码分析

``` c++
//AudioPolicyService.cpp (frameworks\av\services\audiopolicy\service)
void AudioPolicyService::onFirstRef()
{
    {
        ......
        // class AudioPolicyClient : public AudioPolicyClientInterface
        mAudioPolicyClient = new AudioPolicyClient(this);
        mAudioPolicyManager = createAudioPolicyManager(mAudioPolicyClient); //创建Audio管理
    }
}
```

``` c++
//AudioPolicyInterface.h (frameworks\av\services\audiopolicy)
extern "C" AudioPolicyInterface* createAudioPolicyManager(
        AudioPolicyClientInterface *clientInterface)
{
    return new AudioPolicyManager(clientInterface);
}

extern "C" void destroyAudioPolicyManager(AudioPolicyInterface *interface)
{
    delete interface;
}
```

AudioPolicyClientImpl.cpp实现了 AudioPolicyClient，直接与AudioFlinger交互。AudioPolicyIntefaceImpl实现了AudioPolicyInterface直接与AudioPolicyManager交互。

AudioPolicyManager通过解析Audio配置文件，加载各输入输出音频Module，然后遍历各输入输出设备，找出稳定的设备后，由路由引擎设置各设备的连接状态。

``` c++
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
    :
#ifdef AUDIO_POLICY_TEST
    Thread(false),
#endif //AUDIO_POLICY_TEST
    mLimitRingtoneVolume(false), mLastVoiceVolume(-1.0f),
    mA2dpSuspended(false),
    mAudioPortGeneration(1),
    mBeaconMuteRefCount(0),
    mBeaconPlayingRefCount(0),
    mBeaconMuted(false),
    mTtsOutputAvailable(false),
    mMasterMono(false),
    mMusicEffectOutput(AUDIO_IO_HANDLE_NONE)
{
    mpClientInterface = clientInterface;
    bool speakerDrcEnabled = false;
     //解析 xml 得到Audio策略，路由
#ifdef USE_XML_AUDIO_POLICY_CONF
    mVolumeCurves = new VolumeCurvesCollection();
    AudioPolicyConfig config(mHwModules, mAvailableOutputDevices, mAvailableInputDevices,
                             mDefaultOutputDevice, speakerDrcEnabled,
                             static_cast<VolumeCurvesCollection *>(mVolumeCurves));
    if (deserializeAudioPolicyXmlConfig(config) != NO_ERROR) {
#else
    mVolumeCurves = new StreamDescriptorCollection();
    AudioPolicyConfig config(mHwModules, mAvailableOutputDevices, mAvailableInputDevices,
                             mDefaultOutputDevice, speakerDrcEnabled);
    if ((ConfigParsingUtils::loadConfig(AUDIO_POLICY_VENDOR_CONFIG_FILE, config) != NO_ERROR) &&
            (ConfigParsingUtils::loadConfig(AUDIO_POLICY_CONFIG_FILE, config) != NO_ERROR)) {
#endif
        ALOGE("could not load audio policy configuration file, setting defaults");
        config.setDefault();
    }
    // must be done after reading the policy (since conditionned by Speaker Drc Enabling)
    mVolumeCurves->initializeVolumeCurves(speakerDrcEnabled);

    // Once policy config has been parsed, retrieve an instance of the engine and initialize it
    //代理引擎
    audio_policy::EngineInstance *engineInstance = audio_policy::EngineInstance::getInstance();
    if (!engineInstance) {
        ALOGE("%s:  Could not get an instance of policy engine", __FUNCTION__);
        return;
    }
    // Retrieve the Policy Manager Interface
    mEngine = engineInstance->queryInterface<AudioPolicyManagerInterface>(); //获取引擎
    mEngine->setObserver(this); //设为观察者
    status_t status = mEngine->initCheck(); //初始
    (void) status;
    // mAvailableOutputDevices and mAvailableInputDevices now contain all attached devices
    // open all output streams needed to access attached devices
    audio_devices_t outputDeviceTypes = mAvailableOutputDevices.types();
    audio_devices_t inputDeviceTypes = mAvailableInputDevices.types() & ~AUDIO_DEVICE_BIT_IN;
    for (size_t i = 0; i < mHwModules.size(); i++) {
       //调用ClientInterface加载Audio模块，ClientInterface将调用AudioFlinger的loadHwModule
        mHwModules[i]->mHandle = mpClientInterface->loadHwModule(mHwModules[i]->getName());
        if (mHwModules[i]->mHandle == 0) {
            ALOGW("could not open HW module %s", mHwModules[i]->getName());
            continue;
        }
        //查找输出模块设备
        for (size_t j = 0; j < mHwModules[i]->mOutputProfiles.size(); j++)
        {
            const sp<IOProfile> outProfile = mHwModules[i]->mOutputProfiles[j];

            if (!outProfile->hasSupportedDevices()) {
                ALOGW("Output profile contains no device on module %s", mHwModules[i]->getName());
                continue;
            }
            if ((outProfile->getFlags() & AUDIO_OUTPUT_FLAG_TTS) != 0) {
                mTtsOutputAvailable = true;
            }

            if ((outProfile->getFlags() & AUDIO_OUTPUT_FLAG_DIRECT) != 0) {
                continue;
            }
            audio_devices_t profileType = outProfile->getSupportedDevicesType();
            if ((profileType & mDefaultOutputDevice->type()) != AUDIO_DEVICE_NONE) {
                profileType = mDefaultOutputDevice->type();
            } else {
                profileType = outProfile->getSupportedDeviceForType(outputDeviceTypes);
            }
            if ((profileType & outputDeviceTypes) == 0) {
                continue;
            }
            sp<SwAudioOutputDescriptor> outputDesc = new SwAudioOutputDescriptor(outProfile,
                                                                                 mpClientInterface);
            const DeviceVector &supportedDevices = outProfile->getSupportedDevices();
            const DeviceVector &devicesForType = supportedDevices.getDevicesFromType(profileType);
            String8 address = devicesForType.size() > 0 ? devicesForType.itemAt(0)->mAddress
                    : String8("");

            outputDesc->mDevice = profileType;
            audio_config_t config = AUDIO_CONFIG_INITIALIZER;
            config.sample_rate = outputDesc->mSamplingRate;
            config.channel_mask = outputDesc->mChannelMask;
            config.format = outputDesc->mFormat;
            audio_io_handle_t output = AUDIO_IO_HANDLE_NONE;
            //！！！openOutput
            status_t status = mpClientInterface->openOutput(outProfile->getModuleHandle(),
                                                            &output,
                                                            &config,
                                                            &outputDesc->mDevice,
                                                            address,
                                                            &outputDesc->mLatency,
                                                            outputDesc->mFlags);

            if (status != NO_ERROR) {
                ALOGW("Cannot open output stream for device %08x on hw module %s",
                      outputDesc->mDevice,
                      mHwModules[i]->getName());
            } else {
                outputDesc->mSamplingRate = config.sample_rate;
                outputDesc->mChannelMask = config.channel_mask;
                outputDesc->mFormat = config.format;

                for (size_t k = 0; k  < supportedDevices.size(); k++) {
                    ssize_t index = mAvailableOutputDevices.indexOf(supportedDevices[k]);
                    // give a valid ID to an attached device once confirmed it is reachable
                    if (index >= 0 && !mAvailableOutputDevices[index]->isAttached()) {
                        mAvailableOutputDevices[index]->attach(mHwModules[i]);
                    }
                }
                if (mPrimaryOutput == 0 &&
                        outProfile->getFlags() & AUDIO_OUTPUT_FLAG_PRIMARY) {
                    mPrimaryOutput = outputDesc;
                }
                addOutput(output, outputDesc);
                setOutputDevice(outputDesc,
                                outputDesc->mDevice,
                                true,
                                0,
                                NULL,
                                address.string());
            }
        }
        //遍历输入Audio设备
        for (size_t j = 0; j < mHwModules[i]->mInputProfiles.size(); j++)
        {
            const sp<IOProfile> inProfile = mHwModules[i]->mInputProfiles[j];

            if (!inProfile->hasSupportedDevices()) {
                ALOGW("Input profile contains no device on module %s", mHwModules[i]->getName());
                continue;
            }
            // chose first device present in profile's SupportedDevices also part of
            // inputDeviceTypes
            audio_devices_t profileType = inProfile->getSupportedDeviceForType(inputDeviceTypes);

            if ((profileType & inputDeviceTypes) == 0) {
                continue;
            }
            sp<AudioInputDescriptor> inputDesc =
                    new AudioInputDescriptor(inProfile);

            inputDesc->mDevice = profileType;
            ......
            audio_config_t config = AUDIO_CONFIG_INITIALIZER;
            config.sample_rate = inputDesc->mSamplingRate;
            config.channel_mask = inputDesc->mChannelMask;
            config.format = inputDesc->mFormat;
            audio_io_handle_t input = AUDIO_IO_HANDLE_NONE;
            status_t status = mpClientInterface->openInput(inProfile->getModuleHandle(),
                                                           &input,
                                                           &config,
                                                           &inputDesc->mDevice,
                                                           address,
                                                           AUDIO_SOURCE_MIC,
                                                           AUDIO_INPUT_FLAG_NONE);

            if (status == NO_ERROR) {
                const DeviceVector &supportedDevices = inProfile->getSupportedDevices();
                for (size_t k = 0; k  < supportedDevices.size(); k++) {
                    ssize_t index =  mAvailableInputDevices.indexOf(supportedDevices[k]);
                    // give a valid ID to an attached device once confirmed it is reachable
                    if (index >= 0) {
                        sp<DeviceDescriptor> devDesc = mAvailableInputDevices[index];
                        if (!devDesc->isAttached()) {
                            devDesc->attach(mHwModules[i]);
                            devDesc->importAudioPort(inProfile);
                        }
                    }
                }
                mpClientInterface->closeInput(input);
            }
        }
    }
    // make sure all attached devices have been allocated a unique ID
    //遍历输入输出设备，由路由引擎设置各设备的连接状态
    for (size_t i = 0; i  < mAvailableOutputDevices.size();) {
        if (!mAvailableOutputDevices[i]->isAttached()) {
            mAvailableOutputDevices.remove(mAvailableOutputDevices[i]);
            continue;
        }
        // The device is now validated and can be appended to the available devices of the engine
        mEngine->setDeviceConnectionState(mAvailableOutputDevices[i],
                                          AUDIO_POLICY_DEVICE_STATE_AVAILABLE);
        i++;
    }
    for (size_t i = 0; i  < mAvailableInputDevices.size();) {
        if (!mAvailableInputDevices[i]->isAttached()) ;
            mAvailableInputDevices.remove(mAvailableInputDevices[i]);
            continue;
        }
        // The device is now validated and can be appended to the available devices of the engine
        mEngine->setDeviceConnectionState(mAvailableInputDevices[i],
                                          AUDIO_POLICY_DEVICE_STATE_AVAILABLE);
        i++;
    }

    updateDevicesAndOutputs();
}
```

mpClientInterface->openOutput()主要调用是openOutput_l(...)

``` c++
//AudioFlinger.cpp (frameworks\av\services\audioflinger)
sp<AudioFlinger::ThreadBase> AudioFlinger::openOutput_l(.....)
{
    //找到Device, 这个函数很重要，稍后分析
    AudioHwDevice *outHwDev = findSuitableHwDev_l(module, devices);
    //音频流输出到硬件设备
    AudioStreamOut *outputStream = NULL;
    status_t status = outHwDev->openOutputStream(
            &outputStream,
            *output,
            devices,
            flags,
            config,
            address.string());

    mHardwareStatus = AUDIO_HW_IDLE;
}
```

在AudioFlinger createTrack_l()的时候，会去根据流类型来获取路由策略

```  
//Threads.cpp (frameworks\av\services\audioflinger)
uint32_t strategy = AudioSystem::getStrategyForStream(streamType);
        for (size_t i = 0; i < mTracks.size(); ++i) {
            sp<Track> t = mTracks[i];
            if (t != 0 && t->isExternalTrack()) {
            //获取路由策略
                uint32_t actual = AudioSystem::getStrategyForStream(t->streamType());
                if (sessionId == t->sessionId() && strategy != actual) {
                    ALOGE("createTrack_l() mismatched strategy; expected %u but found %u",
                            strategy, actual);
                    lStatus = BAD_VALUE;
                    goto Exit;
                }
            }
        }
```

具体的实现在AudioPolicyInterfaceImpl.cpp文件中
\frameworks\av\services\audiopolicy\service\AudioPolicyInterfaceImpl.cpp,最终将调用Engine.cpp的getStrategyForStream查询音频路由策略

```C++
//Engine.cpp (frameworks\av\services\audiopolicy\enginedefault\src)
routing_strategy Engine::getStrategyForStream(audio_stream_type_t stream)
{
    // stream to strategy mapping
    switch (stream) {
    case AUDIO_STREAM_VOICE_CALL:
    case AUDIO_STREAM_BLUETOOTH_SCO:
        return STRATEGY_PHONE;
    case AUDIO_STREAM_RING:
    case AUDIO_STREAM_ALARM:
        return STRATEGY_SONIFICATION;
    case AUDIO_STREAM_NOTIFICATION:
        return STRATEGY_SONIFICATION_RESPECTFUL;
    case AUDIO_STREAM_DTMF:
        return STRATEGY_DTMF;
    default:
#if !defined(MTK_AUDIO_DEBUG)
        ALOGE("unknown stream type %d", stream);
#endif
    case AUDIO_STREAM_SYSTEM:
        // NOTE: SYSTEM stream uses MEDIA strategy because muting music and switching outputs
        // while key clicks are played produces a poor result
    case AUDIO_STREAM_MUSIC:
        return STRATEGY_MEDIA;
    case AUDIO_STREAM_ENFORCED_AUDIBLE:
        return STRATEGY_ENFORCED_AUDIBLE;
    case AUDIO_STREAM_TTS:
        return STRATEGY_TRANSMITTED_THROUGH_SPEAKER;
    case AUDIO_STREAM_ACCESSIBILITY:
        return STRATEGY_ACCESSIBILITY;
    case AUDIO_STREAM_REROUTING:
        return STRATEGY_REROUTING;
    }
}

```

在AudioPolicyInterface.h文件中定义了如下函数，用来处理设备改变时调整路由策略

``` C++
 virtual status_t handleDeviceConfigChange(audio_devices_t device,
                                              const char *device_address,
                                              const char *device_name) = 0;
```

最终调用到AudioPolicyManager中

``` c++
//AudioPolicyManager.cpp (frameworks\av\services\audiopolicy\managerdefault)

status_t AudioPolicyManager::handleDeviceConfigChange(audio_devices_t device,
                                                      const char *device_address,
                                                      const char *device_name)
{
    status_t status;
    String8 reply;
    AudioParameter param;
    int isReconfigA2dpSupported = 0;

    ALOGV("handleDeviceConfigChange(() device: 0x%X, address %s name %s",
          device, device_address, device_name);

    // connect/disconnect only 1 device at a time
    if (!audio_is_output_device(device) && !audio_is_input_device(device)) return BAD_VALUE;

    // Check if the device is currently connected
    sp<DeviceDescriptor> devDesc =
            mHwModules.getDeviceDescriptor(device, device_address, device_name);
    ssize_t index = mAvailableOutputDevices.indexOf(devDesc);
    if (index < 0) {
        // Nothing to do: device is not connected
        return NO_ERROR;
    }

    // For offloaded A2DP, Hw modules may have the capability to
    // configure codecs. Check if any of the loaded hw modules
    // supports this.
    // If supported, send a set parameter to configure A2DP codecs
    // and return. No need to toggle device state.
    if (device & AUDIO_DEVICE_OUT_ALL_A2DP) {
        reply = mpClientInterface->getParameters(
                    AUDIO_IO_HANDLE_NONE,
                    String8(AudioParameter::keyReconfigA2dpSupported));
        AudioParameter repliedParameters(reply);
        repliedParameters.getInt(
                String8(AudioParameter::keyReconfigA2dpSupported), isReconfigA2dpSupported);
        if (isReconfigA2dpSupported) {
            const String8 key(AudioParameter::keyReconfigA2dp);
            param.add(key, String8("true"));
            mpClientInterface->setParameters(AUDIO_IO_HANDLE_NONE, param.toString());
            return NO_ERROR;
        }
    }

    // Toggle the device state: UNAVAILABLE -> AVAILABLE
    // This will force reading again the device configuration
    status = setDeviceConnectionState(device,
                                      AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE,
                                      device_address, device_name);
    if (status != NO_ERROR) {
        ALOGW("handleDeviceConfigChange() error disabling connection state: %d",
              status);
        return status;
    }

    status = setDeviceConnectionState(device,
                                      AUDIO_POLICY_DEVICE_STATE_AVAILABLE,
                                      device_address, device_name);
    if (status != NO_ERROR) {
        ALOGW("handleDeviceConfigChange() error enabling connection state: %d",
              status);
        return status;
    }

    return NO_ERROR;
}
```



## HAL层

从设计上来看，硬件抽象层是AudioFlinger直接访问的对象。这说明了两个问题，一方面AudioFlinger并不直接调用底层的驱动程序;另一方面，AudioFlinger上层(包括和它同一层的MediaPlayerService)的模块只需要与它进行交互就可以实现音频相关的功能了。因而我们可以认为AudioFlinger是Android音频系统中真正的“隔离板”，无论下面如何变化，上层的实现都可以保持兼容。

音频方面的硬件抽象层主要分为两部分，即AudioFlinger和AudioPolicyService。实际上后者并不是一个真实的设备，只是采用虚拟设备的方式来让厂商可以方便地定制出自己的策略。

抽象层的任务是将AudioFlinger/AudioPolicyService真正地与硬件设备关联起来，但又必须提供灵活的结构来应对变化——特别是对于Android这个更新相当频繁的系统。比如以前Android系统中的Audio系统依赖于ALSA-lib，但后期就变为了tinyalsa，这样的转变不应该对上层造成破坏。因而Audio HAL提供了统一的接口来定义它与AudioFlinger/AudioPolicyService之间的通信方式，这就是audio_hw_device、audio_stream_in及audio_stream_out等等存在的目的，这些Struct数据类型内部大多只是函数指针的定义，是一些“壳”。当AudioFlinger/AudioPolicyService初始化时，它们会去寻找系统中最匹配的实现(这些实现驻留在以audio.primary.*,audio.a2dp.*为名的各种库中)来填充这些“壳”。

根据产品的不同，音频设备存在很大差异，在Android的音频架构中，这些问题都是由HAL层的audio.primary等等库来解决的，而不需要大规模地修改上层实现。换句话说，厂商在定制时的重点就是如何提供这部分库的高效实现了。

## 驱动

Tiny ALSA(Advanced Linux Sound Architecture)音频架构

