---
title: "Mac 打包 FFmpeg二进制文件 双架构流程"
date: 2023-04-14T10:29:09+08:00
draft: false
tags: 
  - iOS
author : winsome
scrolltotop : true
toc : true
mathjax : false
---

首先下载FFmpeg源码 git clone https://github.com/FFmpeg/FFmpeg.git
>ps:这里也有打包好的，不想搞下面的可以直接去这里下载：https://evermeet.cx/ffmpeg/

打包arm64和x86_64需要两个对应环境的brew提供支持。

安装 Homebrew 这个网站写的很详细：https://brew.idayer.com

## 安装 ARM 版 Homebrew
ARM版 Homebrew 最终被安装在/opt/homebrew路径下。

直接执行：/bin/bash -c "$(curl -fsSL https://gitee.com/ineo6/homebrew-install/raw/master/install.sh)"

添加环境：export PATH="/opt/homebrew/bin:$PATH"

## 安装 X86 版 Homebrew
X86版 Homebrew 最终被安装在/usr/local路径下。

执行：arch -x86_64 /bin/bash -c "$(curl -fsSL https://gitee.com/ineo6/homebrew-install/raw/master/install.sh)"

添加环境：export PATH="/usr/local/Homebrew/bin:$PATH"

## 多版本共存
设置别名到环境

alias abrew='arch -arm64 /opt/homebrew/bin/brew'
alias ibrew='arch -x86_64 /usr/local/Homebrew/bin/brew'

abrew、ibrew可以根据你的喜好自定义。

which abrew 应该回来 /opt/homebrew/bin/brew
which ibrew 应该回来 /usr/local/Homebrew/bin/brew

## 切换默认brew
调整对应环境PATH的位置即可。

## 打包 ARM 版 FFmpeg
1. cd到FFmpeg文件夹
2. 执行 ./configure --cc=/usr/bin/clang --prefix=~/Downloads --enable-fontconfig --enable-gpl --enable-version3 --pkg-config-flags=--static --disable-ffplay --extra-cflags=-I/opt/homebrew/include --extra-ldflags=-L/opt/homebrew/lib
3. make clean
4. make -j4

## 打包 X86 版 FFmpeg
1. **终端使用 Rosetta 打开**
2. cd到FFmpeg文件夹
3. 执行 ./configure --cc=/usr/bin/clang --prefix=~/Downloads --enable-fontconfig --enable-gpl --enable-version3 --pkg-config-flags=--static --disable-ffplay --extra-cflags=-I/usr/local/include --extra-ldflags=-L/usr/local/lib
4. make clean
5. make -j4

## 打包说明
1. 打包时可能会报各种文件找不到，用对应的 brew install 即可(abrew install OR brew install)。这个博客写的很好：https://lvv.me/posts/2020/04/14_build_ffmpeg/
2. FFmpeg最后两句指令 --extra-cflags= --extra-ldflags= 表示FFmpeg需要的文件路径，由于来回切换和一些环境因素，可能导致找不到已经下载的库，建议始终加上这两句，需要注意的是 ARM 和 X86 的文件路径不一样。
3. 例子中的 FFmpeg 指令是一些比较基础的指令，根据自己的需求修改。
https://www.cnblogs.com/x_wukong/p/12746031.html
https://blog.csdn.net/u012117034/article/details/128157475
4. 切换打包时，可能报错 does not match previous archive members cputype，直接执行 make clean 即可，为避免出错，例子中每次都加上了这句。

## 合并两个二进制包
lipo -create /Users/**/ffmpeg-x86/ffmpeg /Users/**/ffmpeg-arm/ffmpeg -output /Users/**/ffmpeg-merage/ffmpeg

1. 都是绝对路径。
2. -output 后面跟的路径要加上合成后的包名，要不然会报错。

lipo -info 查看包的架构
