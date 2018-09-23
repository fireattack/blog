---
layout: post
title: Winrar和7z命令行用法
---

## 7z

其实看[官方说明](https://sevenzip.osdn.jp/chm/cmdline/syntax.htm)就OK，但是有几个地方和其他厂家的CLI程序还是挺不一样的，我当时迷糊了半天，于是废话一下。

### 压缩
```
7z.exe a "{output_filename}" "{input_file(s)}" -mmt=4 -m0=LZMA2
```
参数说明：
* a：压缩
* -m：methods，后面直接跟二级选项，无空格
    * -mmt=4：4线程
    * -m0=LZMA2：“第一种”方法指定为LZMA2 （其实是默认值，可以删掉）

输入文件`{input_file(s)}`说明：
* 可以是文件，`file.txt`
* 可以是通配符；但是注意如果要匹配所有文件，用`*`而不要用`*.*`，后者会无法匹配到无后缀的文件（这点和Win默认不一样）
* 可以是文件列表，用@开头，`@filenames.txt`

### 解压缩
```
7z.exe x "{input_filename}" -y -o"{output_path}" 
```
参数说明：
* x：解压缩所有文件（Extract with full paths）
* -y：assume Yes on all queries
* -o：输出目录（`-o`后面直接跟目录名无空格！）

## Winrar 
