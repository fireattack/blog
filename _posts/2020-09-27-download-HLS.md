---
layout: post
title: 下载HLS视频
---

这里推荐3种工具。都顺手帖了下cookie参数的用法，其他参数请参见各个工具的文档。

## FFMPEG

```bat
ffmpeg -headers "Cookie: {cookie_str}" -i {m3u8_url} -c copy output.ts
```

* 可以加`-user_agent`等其他参数，参见[FFMPEG文档](https://ffmpeg.org/ffmpeg-all.html)。
* 加cookie请用上述的`-headers`，不推荐使用`-cookies`：因为他的用法比较复杂，还需要匹配域名和路径。
* `ffmpeg`还支持譬如B站那种的`.flv`的串流，这点是别的很多工具做不到的。顺便一提，B站用非浏览器的UA一般是会被403的，但是我在测试过程中发现如果你用了Postman的默认UA（`PostmanRuntime`）就可以了，都不用伪装成浏览器。

* `ffmpeg`支持master.m3u8，甚至包括音视频分开的串流。但是，`ffmpeg`默认的automatic stream selection是使用分辨率最高的视频和声道最多的音频——如果相同则选用第一个。对于视频一般没问题，但是对于那种音视频分开且有多条不同bitreate的视频的m3u8来说，会选到第一个音轨——一般是码率最低的音轨。我原来从来没注意过这个问题，估计下了很多超低音质的视频ww。

  为了解决这个问题，这是我之前随便写的一个脚本的一部分，仅供参考。就是先用ffprobe把所有流列出来，然后手动筛选出最好的音视频并map。
  
```python
from subprocess import run, check_output

#video_url is your m3u8 address

com = ['ffmpeg', '-user_agent','PostmanRuntime/7.26.1', '-i', video_url, '-c', 'copy']
if video_url.split('?')[0].split('/')[-1].endswith('.m3u8'):
    print('Processing master m3u8...')
    details = check_output(['ffprobe', video_url, '-v', 'quiet', '-show_streams', '-of', 'json']).decode('ansi').strip()
    d = json.loads(details)
    audios = [s for s in d['streams'] if s['codec_type'] == 'audio'] 
    videos = [s for s in d['streams'] if s['codec_type'] == 'video']
    if len(audios) <= 1 and len(videos) <= 1:
        print(f'Only find {len(videos)} video track and {len(audios)} audio track. No mapping needed.')
        pass
    else:
        audios.sort(key=lambda x: int(x.get('bit_rate', 0)))
        for a in audios:
            print(a['index'], a.get('codec_name',''), a.get('bit_rate', 0))
        best_audio_idx = audios[-1]['index']    
        videos.sort(key=lambda x: int(x.get('width',0)))
        for v in videos:
            print(v['index'], v.get('codec_name',''), v.get('width',0), v.get('height',0))
        best_video_idx = videos[-1]['index']
        com.extend(['-map', f'0:{best_video_idx}', '-map', f'0:{best_audio_idx}'])

com.append(filename)
run(com)
```

## [Minyami](https://github.com/Last-Order/Minyami)

```bat
minyami --cookies {cookie_str} -d {m3u8_url} -o output.ts --live
```

* 不支持master.m3u8。

## [Streamlink](https://streamlink.github.io/)

```bat
streamlink --http-header "Cookie={cookie_str}" {m3u8_url} best -o output.ts
```

* Steamlink的后端我记得也是`ffmpeg`。他有个优点是支持对常见直播网站适配，比如B站、twitch之类的。所以你可以选择直接feed进直播间的网址。
* 支持master.m3u8。记得加`best`参数。
* 但是不支持音视频分开的，会只下载视频流。
* 去掉`-o`就是直接播放了。

