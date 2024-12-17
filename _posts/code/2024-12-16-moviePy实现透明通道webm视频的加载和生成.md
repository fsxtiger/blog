---
layout: default
title: moviePy实现透明通道webm视频的加载和生成
---
笔者近期参与开发一个课程制作的项目，在其中负责合成动画相关的模块。这个模块的主要职责是根据输入的元素，如文本、图像和视频，输出指定的视频，在视频中可以为元素定义各种特性，如旋转、模糊、透明等，以及若干种进出场动效效果。笔者经过调研，最终采用moviePy作为动画合成的技术框架，在开发过程中，遇到一个问题：客户端传过来一张图片和一个视频进行动画合成，视频是webm格式，带有透明通道的。由于webm视频背景透明，因此预期结果是以图片作为背景的，内容为webm视频的新视频，然而合成结果背景却是黑色的。为了解决这个问题，笔者比较深入地研究了moviePy框架，将解决过程记录一下。本文首先简要介绍webm视频和moviePy，然后讲解moviePy的一些底层实现，最后给出解决方案和代码。  

webm格式
----
webm是一种媒体文件编码格式，由谷歌在2010年推出，专为网络音视频设计的，特点是具有压缩比高，能够高效支持HTML5音视频标签。笔者对webm格式视频的高压缩比和高效支持HTML5都有直观的感受。透明通道的视频，主流有两种格式：webm和mov。相同内容和时长，webm格式大小仅为mov的0.01左右。至于支持HTML5音视频标签，如下图所示，用chrome浏览器加载webm格式视频，打开代码调试工具将background-color设置为yellow, 视频背景变为黄色。相比之下，mp4不支持透明通道，而mov在chrome中不支持直接渲染播放，可能由于需要占用太多内存的缘故。
<img src="http://glsp-img.unipus.cn/glsp/prod/public/avatar/image/20240204/webm.png" height = "500" width="100%"/>

webm格式视频主要有两类编码器：libvpx-vp8和libvpx-vp9，两种最主要区别是vp9支持透明通道。上图通过设置backgronud-color来实现不同背景颜色的效果，只有vp9编码器能支持。FFmpeg加载webm视频，默认使用vp8编码器。如果使用FFmpeg来加载和生成透明通道的webm视频，需要指定使用vp9编码器，同时使用yuva420像素格式，FFmpeg命令如下：
```
    ffmpeg -c:v libvpx-vp9 -i input.webm -c:v libvpx-vp9 -pix_fmt yuva420p output.webm
```

moviePy框架
----
一句话介绍moviePy，就是将FFmpeg、ImageMagick等工具的能力进行封装，对外提供api，降低开发者学习的门槛。moviePy通过和Numpy、Pillow等库相结合使用，很容易实现一些比较复杂的效果。moviePy正常使用的前提，需要提前安装FFmpeg、ImageMagick等。开发者使用moviePy处理音视频，底层其实是moviePy使用FFmpeg对这些文件进行处理。下图直观展示moviePy和FFmpeg的关系：
<img src="http://glsp-img.unipus.cn/glsp/prod/public/avatar/image/20240204/webm4.png" height = "500" width="100%"/>

屏幕上半部分，是一个运行中的基于moviePy编写的python文件，下半部分打印出和FFmpeg相关的进程。可以很清晰地看到moviePy开启了三个FFmpeg进程，使用ffmpeg -i来加载视频，使用ffmpeg -y 生成视频。moviePy是如何调起FFmpeg进程的呢？这个需要从moviePy的源码说起。moviePy通过VideoFileClip类来加载本地视频文件，通过基类VideoClip的write_videofile方法来生成视频。在这两个方法中，moviePy调起了FFmpeg进程。moviePy中，与FFmpeg交互的逻辑封装在ffmpeg_reader 和 ffmpeg_writer两个模块中。先看下VideoFileClip构造器代码如下：  
```
    VideoClip.__init__(self)
    
    # Make a reader
    pix_fmt = "rgba" if has_mask else "rgb24"
    self.reader = FFMPEG_VideoReader(filename, pix_fmt=pix_fmt,
                                     target_resolution=target_resolution,
                                     resize_algo=resize_algorithm,
                                     fps_source=fps_source)
    
    # Make some of the reader's attributes accessible from the clip
    self.duration = self.reader.duration
    self.end = self.reader.duration
    
    self.fps = self.reader.fps
    self.size = self.reader.size
    self.rotation = self.reader.rotation
    
    self.filename = self.reader.filename
```

在VideoFileClip的构造器中，初始化reader属性，是FFMPEG_VideoReader对象。FFMPEG_VideoReader中通过ffmpeg_parse_infos加载本地视频，读取视频元数据，如时长(duration)，帧率（fps)等。通过initialize方法将本地视频每一帧图像读取到内存进行处理，initialize主要逻辑如下：
```
    if starttime != 0 :
        offset = min(1, starttime)
        i_arg = ['-ss', "%.06f" % (starttime - offset),
            '-i', self.filename,
            '-ss', "%.06f" % offset]
    else:
        i_arg = [ '-i', self.filename]

    
    cmd = ([get_setting("FFMPEG_BINARY")] + i_arg +
               ['-loglevel', 'error',
                '-f', 'image2pipe',
                '-vf', 'scale=%d:%d' % tuple(self.size),
                '-sws_flags', self.resize_algo,
                "-pix_fmt", self.pix_fmt,
                '-vcodec', 'rawvideo', '-'])
    popen_params = {"bufsize": self.bufsize,
                        "stdout": sp.PIPE,
                        "stderr": sp.PIPE,
                        "stdin": DEVNULL}

    if os.name == "nt":
        popen_params["creationflags"] = 0x08000000
        self.proc = sp.Popen(cmd, **popen_params)      
```
同样在执行write_videofile的时候，也会实例化FFMPEG_VideoWriter对象来进行输出，如下所示:
```
    logger = proglog.default_bar_logger(logger)
    if write_logfile:
        logfile = open(filename + ".log", 'w+')
        else:
            logfile = None
        
    logger(message='Moviepy - Writing video %s\n' % filename)
    is_webm = filename.endswith('.webm')
    with WEBM_FFMPEG_VideoWriter(filename, clip.size, fps, codec = codec,
                                preset=preset, bitrate=bitrate, withmask=is_webm, logfile=logfile,
                                audiofile=audiofile, threads=threads,
                                ffmpeg_params=ffmpeg_params) as writer:
    
        nframes = int(clip.duration*fps)
    
        for t,frame in clip.iter_frames(logger=logger, with_times=True,
                                             fps=fps, dtype="uint8"):
            if withmask:
                mask = (255*clip.mask.get_frame(t))
                if mask.dtype != "uint8":
                    mask = mask.astype("uint8")
                    frame = np.dstack([frame,mask])
    
            writer.write_frame(frame)
```

在FFMPEG_VideoReader 和 FFMPEG_VideoWriter类中，开发者就可以覆写相应的方法来实现特定的功能。笔者正是通过覆写FFMPEG_VideoReader中的initialize方法和 FFMPEG_VideoWriter中的ffmpeg_write_video方法来实现透明通道的webm视频加载和生成。同样针对一些FFmpeg支持，但是moviePy没有对外暴露接口的功能，可以通过覆写相应的方法来实现。以重写initialize方法为例。如下所示：
```
    self.close()  # if any

    if starttime != 0:
        offset = min(1, starttime)
        i_arg = ['-ss', "%.06f" % (starttime - offset),
            '-i', self.filename,
            '-ss', "%.06f" % offset]
    else:
        is_webm = self.filename.endswith('.webm')
        if is_webm:
            i_arg = ["-c:v", "libvpx-vp9", '-i', self.filename]
        else:
            i_arg = ['-i', self.filename]

    cmd = ([get_setting("FFMPEG_BINARY")] + i_arg +
        ['-loglevel', 'error',
            '-f', 'image2pipe',
            '-vf', 'scale=%d:%d' % tuple(self.size),
            '-sws_flags', self.resize_algo,
            "-pix_fmt", self.pix_fmt,
            '-vcodec', 'rawvideo', '-'])

    popen_params = {"bufsize": self.bufsize,
                   "stdout": sp.PIPE,
                   "stderr": sp.PIPE,
                   "stdin": DEVNULL}

    if os.name == "nt":
        popen_params["creationflags"] = 0x08000000
        self.proc = sp.Popen(cmd, **popen_params)
```
重新运行python脚本, 如下所示:
<img src="http://glsp-img.unipus.cn/glsp/prod/public/avatar/image/20240204/webm5.png" height = "500" width="100%"/>

可以看到加载webm视频的FFmpeg进程，已经使用libvpx-vp9编码器，生成视频也指定libvpx-vp9编码器。代码已上传[github](https://github.com/fsxtiger/code/blob/master/script/WebmVideoFileClip.py)。加载视频的时候使用WebmVideoFileClip类，合成视频使用WebmCompositeVideoClip类。测试运行正常。