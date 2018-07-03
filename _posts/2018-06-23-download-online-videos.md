---
layout: post
title: 如何下载在线视频
---

一般的用`youtube-dl`就OK。有些m3u8的要手动下可以用ffmpeg (via [SU](https://superuser.com/questions/1260846/downloading-m3u8-videos))：

```
ffmpeg -protocol_whitelist file,http,https,tcp,tls,crypto -i "http://s6.vidshare.tv/hls/pdommq4tlsm4f4kmledsh5d5fcn27i35msjxqw62lfflut5bgaqhb5kirb5q/index-v1-a1.m3u8" -c copy video.mp4
```

这有个Python脚本版：<https://github.com/sam46/TSdownloader>

之前遇到过一个需要下Vimeo的，还要合并音频视频好烦，不过这有个不错的[Python脚本](https://github.com/eMBee/vimeo-download)（Vimeo专用）。使用方法：找到master.json传参进去即可