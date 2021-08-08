---
layout: post
title: FFMPEG常用命令收集
---

## 静态图转BT.709视频

[详细见Blog文](https://fireattack.wordpress.com/2019/09/19/convert-image-to-video-using-ffmpeg/)
```bat
ffmpeg -loop 1 -i colortest_hd.bmp -vf zscale=matrix=709:r=limited,format=yuv420p -color_primaries 1 -color_trc 1 -colorspace 1 -t 30 out2.mp4
```

* `-vf zscale=matrix=709:r=limited`负责转换色域，否则默认转出来是BT.601之类的SD标准。显式指定limited，因为最新版zimg默认是fullrange了。
  * 尽量不要用`scale`，因为有BUG。具体参见上述Blog文
* `-color_primaries 1 -color_trc 1 -colorspace 1`部分负责添加相应的metatag，三者的区别参见[Blog文](https://fireattack.wordpress.com/2018/06/03/topics-about-dvd-encoding/)#Color相关章节
  * x264的话也可以用`-x264opts colorprim=bt709:transfer=bt709:colormatrix=bt709`
* `format=yuv420p`负责输出4:2:0的视频

## 静态图+音频转视频
```bat
ffmpeg -loop 1 -r 1 -i a.png ^
-i a.wav ^
-r 1 -shortest -vf zscale=matrix=709:r=limited,format=yuv420p -c:v libx264 -tune stillimage -y aaa.mp4
```
* 第一行输入1，`-r 1` 来控制input强制1fps读取（否则会把单图重复读取N遍，很慢）；`-loop 1`保证loop（长度无限）。否则后面的`-shortest`无效。~~这里吐个槽，不知道为啥我在FFMPEG的文档完全找不到loop的说明…~~ 找到了，在[ffmpeg-formats.html § 3.9 image2](https://ffmpeg.org/ffmpeg-formats.html])这个image demuxer的说明里。
* 第二行输入2，没有什么要改的。
* 第三行是输出控制，同样`-r 1`减少体积。`-shortest`是选择两个输入最短的（即和音频等长，因为图片我们loop了）。

### 2020-08-02更新

最近发现FFMPEG有个[regression / bug](https://trac.ffmpeg.org/ticket/5456)，使用上述的方式转，出来的视频会比音频长；在input有`-r 1`时更明显（改成较大数值则能缓解）。
在以上ticket里有人推荐最后添加`-fflags +shortest -max_interleave_delta 500M`来缓解，但是在我这里效果不是很明显。有一个笨方法是先压出来，然后再强行`-c copy -t {audio.length}`一波……

## 静态图转Full range视频

参见[Blog文](https://fireattack.wordpress.com/2018/06/09/full-range-video-in-browsers/)

```bat
ffmpeg -loop 1 -i colortest_hd.bmp -vf zscale=matrix=709:r=full,format=yuvj420p -color_primaries 1 -color_trc 1 -colorspace 1 -t 30 out2.mp4
```
* 重点：使用`yuvj420p`。可以显式加r=full或者r=pc，不过好像加不加没区别，yuvj420p已经imply了。
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

无损file level合并 (Concat protocol)

合并TS尤其是HLS下载的TS一定要用这种办法而不要一共上述的steam level（事实上h264之类的都可以先封装ts再直接file level合并），否则会在拼接处出现原始流不存在的卡帧现象（据说是timestamps的锅）。

```bat
ffmpeg -i "concat:1.ts|2.ts|3.ts" -c copy output.ts
```

其实可以直接`copy`：

```bat
copy /b 1.ts + 2.ts + 3.ts output.ts
```

大批量我直接Python：

```python
import shutil

out = open('cat_result.ts','wb')
for f in files: # files is a list of Path obj
    fi = f.open('rb')
    shutil.copyfileobj(fi, out)
    fi.close()
out.close()
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
`-t`是时长，可以替换为`-to`，为终止时间戳。

把`-ss`放在`-i`前和后是[有区别](http://trac.ffmpeg.org/wiki/Seeking)的。先seek（`-ss`在`-i`前面）是input seeking，比较快速。否则是output seeking，遇到大文件相当慢（因为要先读取甚至解码(?)整个文件直到seek的时间）。理论上后者更准确，但是据我观察，不同的seeking方法准确度区别并不大。

同理，`-to/-t`也可以放在`-i`前后。但是，如果放在之前（自然`-ss`也得在之前），这个误差就比较大了，据我观察可能有1、2秒之多，基本不可用。

所以，又快速又相对准确的办法理论上应该是`-ss xxx -i file -to xxx`这个顺序。但是这里有个问题：从output出来，时间戳是会重置回0的，所以`-to`实际会变成类似`-t`的效果（ss 20 to 30会变成20~50秒而不是30~30秒）。所以，要加`-copyts`保持时间戳：`-ss xxx -i file -to xxx -copyts`才行。

然而，**这样也问题**：许多TS文件会有内嵌的start time，copyts的话时间戳会受到start time属性干扰，导致切错时间点（参见[ffmpeg bug ticket](https://trac.ffmpeg.org/ticket/8451)，虽然被关闭了）。

但是`-t`是不受时间戳的变动影响的：所以`-ss xxx -i file -t xxx`（不再需要`-copyts`）依然是准确的。只是，如果想从A时间点切到B，得手动计算下差值了。这里我写了个py脚本帮忙生成命令：

```python
import subprocess
import sys
from pathlib import Path
from django.utils.dateparse import parse_duration

input_file = sys.argv[1]
p = Path(input_file)

p2 = p.with_name(p.stem + ' cut' + p.suffix)
user_input = input('Please input time stamp (press enter to stop")\n')
if user_input == '':
    return
else:
    inputs = user_input.split(' ')
    commands = ['ffmpeg', '-ss', inputs[0], '-i', input_file]            
    if len(inputs) == 2:
        t = str(parse_duration(inputs[1]) - parse_duration(inputs[0]))
        commands.extend(['-t', t])
    commands.extend(['-c', 'copy', str(p2)])
    print('Commands is:', ' '.join(commands))
    subprocess.call(commands)

```

是的，我专门从Django借来了`parse_duration`来计算时间差。不管你信不信，这么基础的功能我居然找不到别的比较robust的库来处理时间戳…随便找了几个都处理不了几个特殊情况。自己手写情况又太多（比如前导〇、毫秒等等一大堆情况需要处理，懒得折腾。）

开了`-c copy`同理，但是注意只能关键帧切割，所以准确度会差一些。

