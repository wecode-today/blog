title: 使用OCRA将ruby脚本转换为EXE可执行文件运行
date: 2014/11/24 9:05:10
categories:
  - ruby
tags:
  - ocra
  - exe
---
__淘汰bat/vbs/js吧！__

OCRA是一个一键式Ruby打包工具，可以将Ruby脚本及所需环境构建为一个单独的Exe文件发布，用户环境上没有Ruby也可以运行！在公司内部作为小工具使用特别方便。

功能：

- LZMA压缩(可选，默认打开)
- 支持Ruby 1.8.7 / 1.9.3 / 2.0.0 ( __通过RubyInstaller安装的版本__ )
- 支持命令行和视窗模式
- 根据使用包含gem包，或者从gemfile包含gem包

OCRA下载：[传送门](http://rubygems.org/gems/ocra)

# 使用介绍

ocra的使用方式很简单，只要安装了ocra之后，执行

```
ocra script.rb
```

即可生成`script.exe`文件，包含`script.rb`文件，Ruby解释器和所有的依赖项（DLL文件和gem包）。

# 实例

对`test.rb`文件打包，生成`print_time.exe`文件：

```
d:\work\open_source\personal_blog\gem_roadmap\problem>ocra D:\temp\test.rb --output D:\temp\print_time.exe
=== Loading script to check dependencies
********************
Time.now = 2014-02-07 16:52:37 +0800
Time.now = 2014-02-07 16:52:37 +0800
Time.now = 2014-02-07 16:52:38 +0800
Time.now = 2014-02-07 16:52:38 +0800
Time.now = 2014-02-07 16:52:38 +0800
Time.now = 2014-02-07 16:52:38 +0800
Time.now = 2014-02-07 16:52:38 +0800
Time.now = 2014-02-07 16:52:39 +0800
Time.now = 2014-02-07 16:52:39 +0800
Time.now = 2014-02-07 16:52:39 +0800
********************
=== Including 53 encoding support files (3194880 bytes, use --no-enc to exclude)
DL is deprecated, please use Fiddle
=== Building D:/temp/print_time.exe
=== Adding user-supplied source files
=== Adding ruby executable ruby.exe
=== Adding detected DLL C:/Ruby200/bin/zlib1.dll
=== Adding detected DLL C:/Ruby200/bin/LIBEAY32.dll
=== Adding detected DLL C:/Ruby200/bin/SSLEAY32.dll
=== Adding detected DLL C:/Ruby200/bin/libffi-6.dll
=== Adding library files
=== Compressing 10006304 bytes
=== Finished building D:/temp/print_time.exe (2518489 bytes)
```

运行后即可生成`print_time.exe`文件，此文件可以在没有安装Ruby的机器中运行。

# **注意事项**
`1.9.3`之后的RubyInstaller不支持`WindowsXP`和`Windows2003`。如果要支持老版本Windows（公司内部很常见），请制作EXE文件时选择1.9.3版本的Ruby。
