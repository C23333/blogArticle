```
title: FFMPEG转换视频格式
date: 2019-08-06 06:12:06
tags: FFmpeg
keywords: FFmpeg
categories: 实操
```



# FFmpeg转换视频格式



* CMD进入视频所在文件夹

* 如果是单个文件，就用如下命令：

  ```text
  ffmpeg -i "input.flv" -c copy "output.mp4"
  ```

  将这里的input改为你的文件名，output改为你想得到的文件名即可。

  如果是整个文件夹中的所有flv文件需要批量转成mp4，那么使用以下命令：

  ```text
  for %i in (*.flv) do ffmpeg -i "%i" -c copy "%~ni.mp4"
  ```

  注：新生成的mp4文件会沿用原文件名。

  