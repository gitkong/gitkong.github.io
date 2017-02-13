---
layout:     post
title:      Audio Queue Services 解读之 Playing Audio(上)
subtitle:    "Audio Queue Services 官方文档翻译"
date:       2017-02-13
author:     gitKong
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - 生活
---

前言：

- 一直想研究一下Audio Queue Services，趁着过年这段时间有空就去研究一下，首选肯定是[官方文档](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW15),下面是我读文档的时候翻译过来，自己一句一句翻译可以加深自己的理解记忆，同时又能方便大家，何乐而不为！
- 由于文档内容较多，本文会分两篇介绍，避免篇幅过长，影响阅读,Demo会迟点分享，敬请期待。

[Audio Queue Services 解读之 Playing Audio(下)](http://www.jianshu.com/p/5d1f466afab1)

# Playing Audio

当你使用 Audio Queue Services 播放音频的时候，音频源可以是任何东西-磁盘文件、基于软件的音频合成、内存中的音频对象等等，这节介绍的是播放磁盘文件

>注意：本章介绍了基于ANSI-C的播放实现，以及一些来自Mac OS X Core Audio SDK的C ++类。 对于基于Objective-C的示例，请参阅 [iOS Dev Center](http://developer.apple.com/devcenter/ios/) 中的SpeakHere示例代码。

为你的应用添加播放功能，你通常需要实现下面步骤：

- **1、定义一个自定义的结构体，管理音频状态、格式和路径等信息。**

- **2、编写一个音频队列回调函数来执行实际播放。**

- **3、编写代码以确定音频队列缓冲区的大小。**

- **4、打开音频文件进行播放并确定其音频数据格式。**

- **5、创建播放音频队列并将其配置为播放。**

- **6、分配和入队音频队列缓冲区， 告诉音频队列开始播放； 完成后，回调会指示音频队列停止。**

- **7、处理音频队列， 释放资源。**

---

# 一、Define a Custom Structure to Manage State

开始之前，先定义一个自定义的结构体，用来管理你的音频样式和音频队列状态信息，例如：

```
static const int kNumberBuffers = 3;                              // 1
struct AQPlayerState {
AudioStreamBasicDescription   mDataFormat;                    // 2
AudioQueueRef                 mQueue;                         // 3
AudioQueueBufferRef           mBuffers[kNumberBuffers];       // 4
AudioFileID                   mAudioFile;                     // 5
UInt32                        bufferByteSize;                 // 6
SInt64                        mCurrentPacket;                 // 7
UInt32                        mNumPacketsToRead;              // 8
AudioStreamPacketDescription  *mPacketDescs;                  // 9
bool                          mIsRunning;                     // 10
};
```

此结构中的大多数字段与用于记录的自定义结构中的字段相同（或接近），如定义管理状态的[Define a Custom Structure to Manage State](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQRecord/RecordingAudio.html#//apple_ref/doc/uid/TP40005343-CH4-SW15)中的录音音频章节所述。 例如，`mDataFormat`字段在此用于保存正在播放的文件的格式。 当记录时，类似字段保存正在写入磁盘的文件的格式。

下面介绍一下结构体中的字段：

- 1、设置audio queue buffers 音频缓存数，三个是最好的，可参考[Audio Queue Buffers](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AboutAudioQueues/AboutAudioQueues.html#//apple_ref/doc/uid/TP40005343-CH5-SW13).

- 2、`AudioStreamBasicDescription `（在`CoreAudioTypes.h`） 代表正在播放的音频数据格式；`mDataFormat`字段通过查询音频文件的`kAudioFilePropertyDataFormat`属性来填充，如[Obtaining a File’s Audio Data Format](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW25)中所述。
有关`AudioStreamBasicDescription`结构的详细信息，请参阅*[Core Audio Data Types Reference](https://developer.apple.com/reference/coreaudio/1613060-core_audio_data_types)*

- 3、你应用创建的播放音频队列对象

- 4、存放缓冲对象 audio queue buffer 的数组

- 5、一个音频文件对象，表示你的程序播放的音频文件。

- 6、每个缓冲对象 audio queue buffer 的大小（以字节为单位），此值在这些示例中在 `DeriveBufferSize` 函数中计算，在音频队列创建后和启动之前。详情看 [Write a Function to Derive Playback Audio Queue Buffer Size](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW23)

- 7、表示要从音频文件播放的下一个数据包的数据包索引。

- 8、在每次调用音频队列的播放回调时要读取的数据包数。 与 `bufferByteSize` 字段一样，在创建音频队列之后和启动音频队列之前，在 `DeriveBufferSize` 函数中的这些示例中计算此值。

- 9、对于VBR（可变比特率）音频数据，正在播放的文件的数据包描述数组。 对于CBR（恒定比特率）数据，此字段的值为NULL。（[VBR 和 CBR 的介绍和区别](http://wenku.baidu.com/link?url=TvnRhJ6CHBz4y8RzE9hg1FQzUepw0YmECQMIsq7eImDlOjBa9i9F5-M9oXf8vf0Mp8SuiAY-3vgkQ2xXXFobMttqBxsIgiFTYHDtyoH_ZVW)）

- 10、表示音频队列是否正在运行

---

# 二、 Write a Playback Audio Queue Callback

下一步就是写一个音频队列 Audio queue 的回调函数，这个函数做了三件事：

- 1、从音频文件读取指定数量的数据，并将其放入音频队列缓冲区。
- 2、将音频队列缓冲区 buffers 加入缓冲区队列 audio queue 中。
- 3、当没有更多的数据要从音频文件读取时，告诉音频队列停止。

此部分显示回调声明示例，分别描述每个任务，最后呈现整个回放回调。 有关回放回调的作用的说明，可以参考下图。

![图](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/Art/playback_callback_function_2x.png)

### (1)、The Playback Audio Queue Callback Declaration

下面展示的 音频队列 播放回调实例声明，在`AudioQueue.h`头文件中声明为`AudioQueueOutputCallback`

```
static void HandleOutputBuffer (
void                 *aqData,                 // 1
AudioQueueRef        inAQ,                    // 2
AudioQueueBufferRef  inBuffer                 // 3
)
```

介绍一下方法参数：

- 1、通常，`aqData ` 是是包含音频队列的状态信息的自定义结构（就上面定义的结构体），看 [Define a Custom Structure to Manage State](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW15) 所述。
- 2、管理这个回调的音频队列
- 3、一个音频队列缓冲区，回调将通过从音频文件读取来填充数据。

### (2)、Reading From a File into an Audio Queue Buffer（从文件中读取到音频队列缓冲区）

回放音频队列回调的第一个动作是从音频文件读取数据并将其放置在音频队列缓冲器中。看下面代码：

```
AudioFileReadPackets (                        // 1
pAqData->mAudioFile,                      // 2
false,                                    // 3
&numBytesReadFromFile,                    // 4
pAqData->mPacketDescs,                    // 5
pAqData->mCurrentPacket,                  // 6
&numPackets,                              // 7
inBuffer->mAudioData                      // 8
);
```

参数说明：

- 1、`AudioFileReadPackets ` 函数声明在 `AudioFile.h` 头文件中，从音频文件中读取数据并放置到音频队列缓冲区中 
- 2、要读取的音频文件
- 3、当读取的时候，用 `false` 来表示函数不缓存数据
- 4、输出时，从音频文件读取的音频数据的字节数
- 5、输出时，从音频文件读取的数据的数据包描述数组。 对于CBR数据，此参数的输入值为NULL
- 6、要从音频文件读取的第一个数据包的数据包索引
- 7、输入时，从音频文件读取的数据包数。 输出时，实际读取的数据包数
- 8、输出时，填充的音频队列缓冲器包含从音频文件读取的数据

### (3)、Enqueuing an Audio Queue Buffer (将音频队列缓冲区 buffers 加入缓冲区队列 audio queue 中)

现在，已经从音频文件读取数据并将其放入音频队列缓冲区中，回调将缓冲区排入队列，如下面代码所示。 一旦进入缓冲器队列，缓冲器中的音频数据就可供音频队列发送到输出设备

```
AudioQueueEnqueueBuffer (                      // 1
pAqData->mQueue,                           // 2
inBuffer,                                  // 3
(pAqData->mPacketDescs ? numPackets : 0),  // 4
pAqData->mPacketDescs                      // 5
);
```

下面介绍一下参数：

- 1、`AudioQueueEnqueueBuffer` 函数将音频队列缓冲区添加到缓冲区队列。
- 2、管理缓冲区队列的音频队列
- 3、要入队的音频队列缓冲区buffers
- 4、音频队列缓冲区数据中表示的数据包数， 对于不使用数据包描述的CBR数据，使用0
- 5、对于使用数据包描述的压缩音频数据格式，缓冲区中数据包的数据包描述

### (4)、Stopping an Audio Queue（停止音频队列）

回调函数的最后一件事是检查是否没有更多的数据要从你正在播放的音频文件中读取。 一旦发现文件的结尾，回调就要告诉播放音频队列停止，看如下代码处理：

```
if (numPackets == 0) {                          // 1
AudioQueueStop (                            // 2
pAqData->mQueue,                        // 3
false                                   // 4
);
pAqData->mIsRunning = false;                // 5
}
```

介绍一下代码：

- 1、通过使用函数 `AudioFileReadPackets` 检查是否有数据包读取（由回调早先调用）
- 2、使用 `AudioQueueStop ` 函数停止音频队列
- 3、要停止的音频队列
- 4、是否马上停止，false的话就当所有排队缓冲区都已播放时，异步停止音频队列。
- 5、重置音频队列状态不是正在运行

### (5)、A Full Playback Audio Queue Callback（完整的回调函数）

```
static void HandleOutputBuffer (
void                *aqData,
AudioQueueRef       inAQ,
AudioQueueBufferRef inBuffer
) {
// 官方文档这里写的有问题
//AQPlayerState *pAqData = (AQPlayerState *) aqData;        // 1
struct AQPlayerState *pAqData = aqData;
if (pAqData->mIsRunning == 0) return;                     // 2
UInt32 numBytesReadFromFile;                              // 3
UInt32 numPackets = pAqData->mNumPacketsToRead;           // 4
AudioFileReadPackets (
pAqData->mAudioFile,
false,
&numBytesReadFromFile,
pAqData->mPacketDescs, 
pAqData->mCurrentPacket,
&numPackets,
inBuffer->mAudioData 
);
if (numPackets > 0) {                                     // 5
inBuffer->mAudioDataByteSize = numBytesReadFromFile;  // 6
AudioQueueEnqueueBuffer ( 
pAqData->mQueue,
inBuffer,
(pAqData->mPacketDescs ? numPackets : 0),
pAqData->mPacketDescs
);
pAqData->mCurrentPacket += numPackets;                // 7 
} else {
AudioQueueStop (
pAqData->mQueue,
false
);
pAqData->mIsRunning = false; 
}
}
```

部分代码介绍：

- 1、在实例化时提供给音频队列的定制数据，包括表示要播放的文件的音频文件对象（AudioFileID类型）以及各种状态数据。 请参阅[Define a Custom Structure to Manage State](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW15)。
- 2、如果音频队列停止，则立即返回。
- 3、保存从正在播放的文件中读取的音频数据的字节数的变量。
- 4、使用从正在播放的文件中读取的数据包数量初始化 `numPackets` 变量。
- 5、测试是否从文件中检索了某些音频数据。 如果是，则将新填充的缓冲区排入队列。 如果不是，停止音频队列。
- 6、告诉音频队列缓冲区结构读取的数据的字节数。
- 7、根据读取的数据包数量增加数据包索引。

---

# 三、Write a Function to Derive Playback Audio Queue Buffer Size（编写一个函数去获取播放音频队列缓冲区的大小）

Audio Queue Services 希望你在应用里面给你的音频队列缓冲区指定大小，下面提供的代码可以导出足够大的缓冲器大小以容纳给定持续时间的音频数据。

创建音频队列后，可以通过调用 `DeriveBufferSize `(下面的方法)来作为请求音频队列分配缓冲区的先决条件。可查看 [Set Sizes for a Playback Audio Queue](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW9)

这里的代码做了两个额外的事情，对比 [Write a Function to Derive Recording Audio Queue Buffer Size](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQRecord/RecordingAudio.html#//apple_ref/doc/uid/TP40005343-CH4-SW14) 类似的

- 1、导出每次回调调用 `AudioFileReadPackets` 函数时要读取的数据包数
- 2、设置缓冲区大小的下限，以避免磁盘访问频率过高

这里的计算考虑了从磁盘读取的音频数据格式。 格式包括可能影响缓冲区大小的所有因素，如音频通道数

```
void DeriveBufferSize (
// 不应该有&
//AudioStreamBasicDescription &ASBDesc,                            // 1
AudioStreamBasicDescription ASBDesc,
UInt32                      maxPacketSize,                       // 2
Float64                     seconds,                             // 3
UInt32                      *outBufferSize,                      // 4
UInt32                      *outNumPacketsToRead                 // 5
) {
static const int maxBufferSize = 0x50000;                        // 6
static const int minBufferSize = 0x4000;                         // 7

if (ASBDesc.mFramesPerPacket != 0) {                             // 8
Float64 numPacketsForTime =
ASBDesc.mSampleRate / ASBDesc.mFramesPerPacket * seconds;
*outBufferSize = numPacketsForTime * maxPacketSize;
} else {                                                         // 9
*outBufferSize =
maxBufferSize > maxPacketSize ?
maxBufferSize : maxPacketSize;
}

if (                                                             // 10
*outBufferSize > maxBufferSize &&
*outBufferSize > maxPacketSize
)
*outBufferSize = maxBufferSize;
else {                                                           // 11
if (*outBufferSize < minBufferSize)
*outBufferSize = minBufferSize;
}

*outNumPacketsToRead = *outBufferSize / maxPacketSize;           // 12
}
```

代码介绍：

- 1、音频队列的AudioStreamBasicDescription结构。
- 2、播放的音频文件中数据的估计最大包大小。可以通过调用属性ID为 `kAudioFilePropertyPacketSizeUpperBound` 的 `AudioFileGetProperty` 函数（在 `AudioFile.h` 头文件中声明）来确定此值。请参阅[Set Sizes for a Playback Audio Queue](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW9)。
- 3、为每个音频队列缓冲区指定的大小（以秒为单位）。
- 4、输出时，每个音频队列缓冲区的大小（以字节为单位）。
- 5、输出时，在每次调用回放音频队列回调时从文件读取的音频数据的数据包数。
- 6、音频队列缓冲区大小的上限（以字节为单位）。在此示例中，上限设置为320 KB。这对应于大约5秒的立体声，24位音频，采样率为96kHz。
- 7、音频队列缓冲区大小的下限（以字节为单位）。在此示例中，下限设置为16 KB。
- 8、对于定义每个分组固定数量的帧的音频数据格式，导出音频队列缓冲区大小。
- 9、对于没有为每个分组定义固定数量的帧的音频数据格式，基于最大分组大小和你设置的上限，导出合理的音频队列缓冲区大小。
- 10、如果导出的缓冲区大小高于你设置的上限，请根据估计的最大数据包大小调整绑定。
- 11、如果派生的缓冲区大小低于你设置的下限，请将其调整到绑定。
- 12、计算在每次调用回调时从音频文件读取的数据包数。

---

# 四、Open an Audio File for Playback（打开音频文件播放）

现在播放音频文件只需要下面三个步骤：

- 1、获取表示要播放的音频文件的 `CFURL` 对象
- 2、打开文件
- 3、获取文件的音频数据格式

### (1)、Obtaining a CFURL Object for an Audio File（获取播放文件的CFURL对象）

通过下面代码获取：
```
CFURLRef audioFileURL =
CFURLCreateFromFileSystemRepresentation (           // 1
NULL,                                           // 2
(const UInt8 *) filePath,                       // 3
strlen (filePath),                              // 4
false                                           // 5
);
```



代码介绍：

- 1、在 `CFURL.h` 头文件中声明的 `CFURLCreateFromFileSystemRepresentation` 函数创建一个表示要播放的文件的CFURL对象。
- 2、使用 `NULL`（或 `kCFAllocatorDefault`）来使用当前的默认内存分配器。
- 3、要转换为CFURL对象的文件系统路径。 在生产代码中，通常从用户获取filePath的值。
- 4、文件系统路径中的字节数。
- 5、值为false表示filePath表示文件，而不是目录。


**还有一种方法，我觉得是比较常用的，在我demo就使用这个，这个是通过传入一个NSString 路径实现的**

```
CFStringRef strRef = (__bridge CFStringRef)filePath;
// CFURLPathStyle 不建议使用kCFURLHFSPathStyle。 使用HFS样式路径的Carbon文件管理器已被弃用。 HFS样式路径不可靠，因为它们可以随意引用多个卷（如果这些卷具有相同的卷名称）。 您应该尽可能使用kCFURLPOSIXPathStyle。
CFURLRef audioFileURL =
CFURLCreateWithFileSystemPath(NULL,
strRef,
kCFURLPOSIXPathStyle,
YES
);
```

- 其中CFURLPathStyle 不建议使用kCFURLHFSPathStyle。 使用HFS样式路径的Carbon文件管理器已被弃用。 HFS样式路径不可靠，因为它们可以随意引用多个卷（如果这些卷具有相同的卷名称）。 官方建议应该尽可能使用这个


### (2)、Opening an Audio File（打开音频文件）

下面示例演示怎么去打开一个音频文件去播放

```
AQPlayerState aqData;                                   // 1

OSStatus result =
AudioFileOpenURL (                                  // 2
audioFileURL,                                   // 3
fsRdPerm,                                       // 4
0,                                              // 5
&aqData.mAudioFile                              // 6
);

CFRelease (audioFileURL); 
```

代码解释：

- 1、创建一个 `AQPlayerState ` 自定义结构体实例（可查看 [Define a Custom Structure to Manage State](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW15)），当你打开一个音频文件播放的时候，这个实例可以控制正在播放的音频文件（类型是 `AudioFileID `）
- 2、`AudioFileOpenURL ` 函数声明在 `AudioFile.h  ` 头文件，打开一个你想播放的音频文件
- 3、`CFURLRef` 播放引用url
- 4、你播放的音频文件所需的文件权限，可用的文件权限定义在文件管理 `File Access Permission Constants` 枚举中
- 5、可选的文件类型提示。 值为0表示该示例不使用此功能
- 6、在输出上，对音频文件的引用被放置在自定义结构的`mAudioFile`字段中
- 7、释放在步骤1中创建的`CFURLRef` 对象

### (3)、Obtaining a File’s Audio Data Format（获取文件的音频数据格式）

上代码：

```
UInt32 dataFormatSize = sizeof (aqData.mDataFormat);    // 1

AudioFileGetProperty (                                  // 2
aqData.mAudioFile,                                  // 3
kAudioFilePropertyDataFormat,                       // 4
&dataFormatSize,                                    // 5
&aqData.mDataFormat                                 // 6
);
```

- 1、查询音频文件的音频数据格式的时候获取预期值
- 2、`AudioFileGetProperty` 函数，声明在 `AudioFile.h` 头文件中，获取指定音频文件中属性的值
- 3、你想要获取的音频数据格式的音频文件对象，类型是 `AudioFileID`
- 4、获取音频文件数据格式值的属性ID
- 5、在输入时，描述音频文件的数据格式的`AudioStreamBasicDescription`结构体的预期大小；在输出时，实际大小， 播放应用程序不需要使用此值
- 6、在输出时，完整的音频数据格式，以`AudioStreamBasicDescription`结构体的形式，从音频文件获得。 此行通过将文件的音频数据格式存储在音频队列的自定义结构中来将其应用于音频队列。

---

#### 下篇将介绍 创建音频播放队列并实现播放，会附上Demo，前往：[Audio Queue Services 解读之 Playing Audio(下)](http://www.jianshu.com/p/5d1f466afab1)

---

#### **欢迎大家关注我，喜欢就点个like和star，你的支持将是我的动力~**

>翻译过来的可能有出入，如果大家发现有什么问题或者写错的，欢迎留言，谢谢

