---
layout:     post
title:      Audio Queue Services 解读之 Playing Audio(下)
subtitle:    "Audio Queue Services 官方文档翻译"
date:       2017-02-13
author:     gitKong
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - 生活
---

**解读Play Audio下集，如果你没看上集，建议先去看看上集.**

[Audio Queue Services 解读之 Playing Audio(上)](http://www.jianshu.com/p/d2ef4d15356c)

>**上集已经准备好音频队列的结构体以及回调函数，那么接下来就可以创建音频队列并实现播放了！**

# 五、Create a Playback Audio Queue（创建播放的音频队列）

下面将演示怎么去创建回放音频队列，注意：`AudioQueueNewOutput ` 函数中使用的自定义结构体和回调方法都是在之前写好了

```
AudioQueueNewOutput (                                // 1
&aqData.mDataFormat,                             // 2
HandleOutputBuffer,                              // 3
&aqData,                                         // 4
CFRunLoopGetCurrent (),                          // 5
kCFRunLoopCommonModes,                           // 6
0,                                               // 7
&aqData.mQueue                                   // 8
);
```

- 1、通过 `AudioQueueNewOutput` 函数来创建一个回放音频队列
- 2、需要设置播放的音频队列对应的音频文件数据格式，参考[Obtaining a File’s Audio Data Format](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW25)
- 3、回放音频队列的回调函数，参考[Write a Playback Audio Queue Callback](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW2)
- 4、音频队列对应的自定义结构体，可查看[Define a Custom Structure to Manage State](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW15)
- 5、当前运行循环，以及将调用音频队列回放回调的循环
- 6、当前可以被调用回调的运行循环模式，通常使用 `kCFRunLoopCommonModes `
- 7、保留字段，传0
- 8、输出时，新分配的回放音频队列


# 六、Set Sizes for a Playback Audio Queue（设置播放音频队列的大小）

现在，你可以为播放音频队列设置一些大小，当你给缓冲队列分配缓冲区的时候和开始读取音频文件之前，你可以使用这些设置的大小

下面代码将会告诉你怎么设置

- 1、音频队列缓冲区大小
- 2、每次执行回放音频队列回调要读取的数据包个数
- 3、用于保存一个缓冲区的音频数据的分组描述的数组大小

### (1)、Setting Buffer Size and Number of Packets to Read（设置缓冲区大小和要读取的数据包数）

下面代码段会演示如何使用之前写的 `DeriveBufferSize ` 函数（see [Write a Function to Derive Playback Audio Queue Buffer Size](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW23)）这里是为每一个音频缓冲区设置一个大小，单位是字节；以及确认每次调用回放音频回调要读取的数据包数

此代码使用最大包大小的保守估计，Core Audio通过 `kAudioFilePropertyPacketSizeUpperBound` 属性提供。 在大多数情况下，最好使用这种方式 - 这是近似但快速的 - 花费时间读取整个音频文件以获得实际的最大包大小

```
UInt32 maxPacketSize;
UInt32 propertySize = sizeof (maxPacketSize);
AudioFileGetProperty (                               // 1
aqData.mAudioFile,                               // 2
kAudioFilePropertyPacketSizeUpperBound,          // 3
&propertySize,                                   // 4
&maxPacketSize                                   // 5
);

DeriveBufferSize (                                   // 6
aqData.mDataFormat,                              // 7
maxPacketSize,                                   // 8
0.5,                                             // 9
&aqData.bufferByteSize,                          // 10
&aqData.mNumPacketsToRead                        // 11
);
```

下面介绍一下代码的作用：

- 1、`AudioFileGetProperty ` 函数，声明在 `AudioFile.h` 头文件中，可以获取音频文件中指定属性的值，这里你可以用来获取一个你要播放的音频文件的音频数据包大小的保守上限
- 2、音频文件对象（类型是 `AudioFileID`）代表你要播放的音频文件，可查看[Opening an Audio File](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW24)
- 3、属性ID，用于在音频文件中获取数据包大小的保守上限
- 4、输出时，kAudioFilePropertyPacketSizeUpperBound属性的大小（以字节为单位）
- 5、输出是，你要播放的音频文件数据包大小的保守上限（以字节为单位）
- 6、`DeriveBufferSize ` 函数，声明在[Write a Function to Derive Playback Audio Queue Buffer Size](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW23)，可以设置缓冲区大小和每次调用回调要读取的数据包数
- 7、你要播放的文件数据格式，可查看 [Obtaining a File’s Audio Data Format](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW25)
- 8、音频文件预估最大的数据包大小，上面第5点提到那个
- 9、每个音频队列缓冲区应该保持的音频的秒数，通常0.5秒是一个很好的选择
- 10、输出时、每个音频队列缓冲区的大小（以字节为单位），这个值存放在音频队列的自定义结构体中
- 11、输出时，每次执行回调要读取的数据包数，这个值也是存储在音频队列的自定义结构体中

### (2)、Allocating Memory for a Packet Descriptions Array（为包描述数组分配内存）

现在为数组分配内存，以保存一个缓冲区的音频数据的数据包描述。 恒定比特率数据不使用数据包描述，因此下面代码中CBR情形 - 步骤3非常简单。

```
bool isFormatVBR = (                                       // 1
aqData.mDataFormat.mBytesPerPacket == 0 ||
aqData.mDataFormat.mFramesPerPacket == 0
);

if (isFormatVBR) {                                         // 2
aqData.mPacketDescs =
(AudioStreamPacketDescription*) malloc (
aqData.mNumPacketsToRead * sizeof (AudioStreamPacketDescription)
);
} else {                                                   // 3
aqData.mPacketDescs = NULL;
}
```

下面介绍代码作用

- 1、确定音频文件的数据格式是VBR还是CBR。 在VBR数据中，每个字节的字节数或每帧的帧数是可变的，因此将在音频队列的AudioStreamBasicDescription结构中列为0。
- 2、对于包含VBR数据的音频文件，为包描述数组分配内存。 基于在每次调用回放回调时要读取的音频数据包的数量来计算所需的内存。 请参阅[Setting Buffer Size and Number of Packets to Read](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW26)。
- 3、对于包含CBR数据的音频文件（例如线性PCM），音频队列不使用包描述数组。

# 七、Set a Magic Cookie for a Playback Audio Queue（设置回放音频队列的 Magic Cookie ）

一些音频压缩的音频格式，例如 MPEG 4 AAC，利用结构体包含音频的元数据。这些结构体就是Magic Cookie，当你用 Audio Queue Services 播放这种格式的音频文件时，你可以从音频文件中获取Magic Cookie ，然后在播放之前添加到音频队列中

下面代码教你怎么去从音频文件中获取Magic Cookie，并添加到音频队列中，下面的代码需要在开始回放之前执行


```
UInt32 cookieSize = sizeof (UInt32);                   // 1
bool couldNotGetProperty =                             // 2
AudioFileGetPropertyInfo (                         // 3
aqData.mAudioFile,                             // 4
kAudioFilePropertyMagicCookieData,             // 5
&cookieSize,                                   // 6
NULL                                           // 7
);

if (!couldNotGetProperty && cookieSize) {              // 8
char* magicCookie =
(char *) malloc (cookieSize);

AudioFileGetProperty (                             // 9
aqData.mAudioFile,                             // 10
kAudioFilePropertyMagicCookieData,             // 11
&cookieSize,                                   // 12
magicCookie                                    // 13
);

AudioQueueSetProperty (                            // 14
aqData.mQueue,                                 // 15
kAudioQueueProperty_MagicCookie,               // 16
magicCookie,                                   // 17
cookieSize                                     // 18
);

free (magicCookie);                                // 19
}
```

下面介绍一下代码的作用：

- 1、设置magic cookie 的预估大小
- 2、获取函数 `AudioFileGetPropertyInfo ` 的返回值，如果成功，返回 `NoErr`,相当于布尔值 `false`
- 3、函数 `AudioFileGetPropertyInfo ` 声明在 `AudioFile.h` 头文件，可以获取指定属性值的大小，可以使用它来设置保存属性值的变量的大小
- 4、你要播放的音频文件对象，类型是 `AudioFileID`
- 5、属性ID 表示音频文件的 Magic Cookie 数据
- 6、输入时，是magic cookie 数据的预估大小；输出时，是真实的大小
- 7、使用 `NULL` 表示你不关心这个属性的读写访问
- 8、如果音频文件包含有 magic cookie，那么就为它分配内存进行管理
- 9、函数 `AudioFileGetProperty ` 声明在 `AudioFile.h` 头文件，可以获取指定属性值，在这里是获取音频文件的 magic cookie
- 10、表示你要播放的并且获取到magic cookie的音频文件对象，类型是 `AudioFileID`
- 11、属性ID 表示音频文件的 Magic Cookie 数据
- 12、输入时，通过使用函数 `AudioFileGetPropertyInfo ` 获取 `magicCookie ` 变量的大小；输出时，真实的magic cookie 大小就是写入到 `magicCookie ` 变量的字节数
- 13、输出时，音频文件的 magic cookie
- 14、函数 `AudioQueueSetProperty ` 设置音频队列的属性，在这里，用来匹配要被播放的音频文件的magic cookie并设置给音频队列
- 15、你要设置magic cookie 的音频队列
- 16、属性ID 表示音频文件的 Magic Cookie 数据
- 17、你要播放的音频文件的 magic cookie
- 18、magic cookie 的大小，单位是字节
- 19、释放之前被分配内存的magic cookie

# 八、Allocate and Prime Audio Queue Buffers（分配和填充音频队列缓冲区）

现在你可以让你之前创建的音频对象去准备一些音频队列缓冲区，下面代码教你怎么做

```
aqData.mCurrentPacket = 0;                                // 1

for (int i = 0; i < kNumberBuffers; ++i) {                // 2
AudioQueueAllocateBuffer (                            // 3
aqData.mQueue,                                    // 4
aqData.bufferByteSize,                            // 5
&aqData.mBuffers[i]                               // 6
);

HandleOutputBuffer (                                  // 7
&aqData,                                          // 8
aqData.mQueue,                                    // 9
aqData.mBuffers[i]                                // 10
);
}
```

下面介绍代码的作用：

- 1、将包索引设置为0，以便在音频队列回调开始填充缓冲区时（步骤7），它从音频文件的开头开始
- 2、分配和填充音频队列缓冲区（你在 [Define a Custom Structure to Manage State](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW15) 设置了 `kNumberBuffers `这个值为 `3`）
- 3、函数 `AudioQueueAllocateBuffer ` 通过分配内存来创建音频队列缓冲区
- 4、负责分配音频队列缓冲区的音频队列
- 5、新的音频队列缓冲区的大小，单位是字节
- 6、输出时，添加新的音频队列缓冲区到自定义结构体的 `mBuffers` 数组中
- 7、函数 `HandleOutputBuffer ` 是你之前写的回放音频队列回调，可查看 [Write a Playback Audio Queue Callback](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AQPlayback/PlayingAudio.html#//apple_ref/doc/uid/TP40005343-CH3-SW2)
- 8、音频队列的自定义结构体
- 9、调用回调的音频队列
- 10、传递到音频队列回调的音频队列缓冲区

# 九、Set an Audio Queue’s Playback Gain（设置音频队列回放增益）

在告诉音频队列播放之前，你可以通过音频队列参数机制设置其增益，下面代码教你怎么设置，更多设置可以查看 [Audio Queue Parameters](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/AboutAudioQueues/AboutAudioQueues.html#//apple_ref/doc/uid/TP40005343-CH5-SW15)


```
Float32 gain = 1.0;                                       // 1
// Optionally, allow user to override gain setting here
AudioQueueSetParameter (                                  // 2
aqData.mQueue,                                        // 3
kAudioQueueParam_Volume,                              // 4
gain                                                  // 5
);
```

下面介绍一下代码的作用：

- 1、设置音频队列的增益，在0-1之间
- 2、通过函数 `AudioQueueSetParameter ` 设置音频队列的参数值
- 3、设置参数得音频队列
- 4、设置参数的ID，kAudioQueueParam_Volume常数可让你设置音频队列的增益
- 5、应用于音频队列的增益设置

# 十、Start and Run an Audio Queue（开启并运行音频队列）

所有上述代码已经播放文件的过程。 这包括在播放文件时启动音频队列并维护运行循环，看下面代码

```
aqData.mIsRunning = true;                          // 1

AudioQueueStart (                                  // 2
aqData.mQueue,                                 // 3
NULL                                           // 4
);

do {                                               // 5
CFRunLoopRunInMode (                           // 6
kCFRunLoopDefaultMode,                     // 7
0.25,                                      // 8
false                                      // 9
);
} while (aqData.mIsRunning);

CFRunLoopRunInMode (                               // 10
kCFRunLoopDefaultMode,
1,
false
);
```

介绍代码作用：

- 1、设置自定义结构体的标识，指示音频队列正在运行
- 2、使用函数 `AudioQueueStart ` 开启音频队列，在自己的线程中
- 3、要开启的音频队列
- 4、使用 `NULL` 表示音频队列需要马上开启播放
- 5、定期轮询自定义结构的mIsRunning字段，以检查音频队列是否已停止
- 6、`CFRunLoopRunInMode` 函数运行包含音频队列的线程的运行循环
- 7、使用默认的运行循环模式
- 8、设置运行循环的运行时间为0.25秒
- 9、使用 `false` 表示运行循环应该继续指定整个时间
- 10、在音频队列停止后，再运行循环一会儿，以确保当前播放的音频队列缓冲区有时间完成


# 十一、Clean Up After Playing（播放完毕后清除）

当你的音频文件播放完毕，应该要销毁这个音频队列，关闭音频文件，同时要释放所有资源，下面代码处理了这些步骤

```
AudioQueueDispose (                            // 1
aqData.mQueue,                             // 2
true                                       // 3
);

AudioFileClose (aqData.mAudioFile);            // 4

free (aqData.mPacketDescs);                    // 5
```

下面介绍一下代码的作用：

- 1、函数 `AudioQueueDispose ` 销毁音频队列和它的所有资源，包括它的缓冲区
- 2、要销毁的音频队列
- 3、使用true可以同步销毁音频队列
- 4、关闭播放完毕的音频文件，函数 `AudioFileClose ` 声明在 `AudioFile.h` 头文件中
- 5、释放用于保存数据包描述的内存

# 十二、总结： 

- 1、实现上面的步骤，就可以播放本地的音频文件了，后续工作就是优化封装了
- 2、因为上面步骤只能实现播放本地音频文件，至于网络音频文件播放就没提，后续 出的Demo 会实现播放网络音频文件，敬请期待
- 3、playing audio 这篇的翻译算是完成了，欢迎大家去简书关注我，喜欢就给个like，如果有问题，请留言喔，谢谢
