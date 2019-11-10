---
layout: post
title: FFMPEG常用命令收集
---

## 静态图转BT.709视频

[详细见Blog文](https://fireattack.wordpress.com/2019/09/19/convert-image-to-video-using-ffmpeg/)
```bat
ffmpeg -loop 1 -i colortest_hd.bmp -vf zscale=matrix=709,format=yuv420p -color_primaries 1 -color_trc 1 -colorspace 1 -t 30 out2.mp4
```

* `-vf zscale=matrix=709`负责转换色域，否则默认转出来是BT.601之类的SD标准
  * 尽量不要用`scale`，因为有BUG。具体参见上述Blog文
* `-color_primaries 1 -color_trc 1 -colorspace 1`部分负责添加相应的metatag，三者的区别参见[Blog文](https://fireattack.wordpress.com/2018/06/03/topics-about-dvd-encoding/)#Color相关章节
  * x264的话也可以用`-x264opts colorprim=bt709:transfer=bt709:colormatrix=bt709`
* `format=yuv420p`负责输出4:2:0的视频

## 静态图转Full range视频

参见[Blog文](https://fireattack.wordpress.com/2018/06/09/full-range-video-in-browsers/)

```bat
ffmpeg -loop 1 -i colortest_hd.bmp -vf zscale=matrix=709,format=yuvj420p -color_primaries 1 -color_trc 1 -colorspace 1 -t 30 out2.mp4
```
* 重点：使用`yuvj420p`
* `-color_range 2`不要用，效果其实就是强行在元数据里塞个full range，结果视频的像素数值还都是16-235范围内的，也就是说会出来一个灰暗的色域未正常伸张的视频
* 也有`-x264opts fullrange=on`，没试过

## 下载m3u8

(via [SU](https://superuser.com/questions/1260846/downloading-m3u8-videos))：

```bat
ffmpeg -protocol_whitelist file,http,https,tcp,tls,crypto -i "http://s6.vidshare.tv/hls/pdommq4tlsm4f4kmledsh5d5fcn27i35msjxqw62lfflut5bgaqhb5kirb5q/index-v1-a1.m3u8" -c copy video.mp4
```

* 如果m3u8是一个本地文件，前面的`-protocol_whitelist`的部分是必须的。否则无必要。

## 合并视频

[官方教程](https://trac.ffmpeg.org/wiki/Concatenate)

无损stream level合并（比如合并B站分P的）：

```bat
(for %i in (*.flv) do @echo file '%i') > mylist.txt
ffmpeg -f concat -safe 0 -i mylist.txt -c copy output.mp4
```

如果是直接重编码的合并：

```bat
ffmpeg -i input1.mp4 -i input2.webm \
-filter_complex "[0:v:0] [0:a:0] [1:v:0] [1:a:0] concat=n=2:v=1:a=1 [v] [a]" \
-map "[v]" -map "[a]" <encoding options> output.mkv
```

多文件范例：

```bat
ffmpeg -i 001_edit.mkv -i 002_edit.mkv -i 003_edit.mkv -i 004_edit.mkv -i 005_edit.mkv -i 006_edit.mkv -i 007_edit.mkv -filter_complex "[0:v:0] [0:a:0] [1:v:0] [1:a:0] [2:v:0] [2:a:0] [3:v:0] [3:a:0] [4:v:0] [4:a:0] [5:v:0] [5:a:0] [6:v:0] [6:a:0] concat=n=7:v=1:a=1 [v] [a]" -map "[v]" -map "[a]" -c:v libx264 -preset slower -crf 18 -profile:v high -level 5.0 -c:a libvorbis -qscale:a 6 result_final.mp4
```

## 切割视频

```bat
ffmpeg -i input.mkv -ss 00:00:00 -t 00:04:13.940 -c:v libx264 -preset fast -crf 18 -c:a flac cut.mkv
```
* `-t`是时长，可以替换为`-to`为截止时间戳。
* 把`-i`放在`-ss`/`-to`前和后有区别，放在前面seek比较快，但是不准，推荐还是放后面。
* `-c copy`同理，但是注意只能关键帧切割，所以不一定准。

