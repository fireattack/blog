---
layout: post
title: ExifTool常用命令
---

资料：
* [官方文档](https://www.sno.phy.queensu.ca/~phil/exiftool/exiftool_pod.html)

### 复制所有元数据

```
exiftool -TagsFromFile src.jpg dst.jpg
```

或者

```
exiftool -TagsFromFile src.jpg -all:all dst.jpg
```

两者的区别相当微妙，似乎是前者会优先把tag写到（更）合适的group里，后者则会完全保留src的group。

### 从src到dst复制ICC

```
exiftool -TagsFromFile src.jpg -icc_profile dst.jpg
```

上面那个默认是不会复制icc的，所以要这样。也可以一起：

```
exiftool -TagsFromFile src.jpg -all:all -icc_profile dst.jpg
```

### 从image提取ICC

```
exiftool -icc_profile -b -w icc image.jpg
```

### 添加ICC到图片image

```
exiftool "-ICC_Profile<=D:\AdobeRGB1998.icc" image.jpg
```
