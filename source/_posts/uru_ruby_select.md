title: 在Windows下使用uru用于多个ruby的切换
date: 2014/11/24 9:05:00
categories:
  - ruby
tags:
  - rbenv
  - ruby
---

## Ruby选择器

你还在为如何使用系统中的各个版本的Ruby而烦恼吗？Uru正是为你设计的。

## 如何使用

很简单。

- 在C盘根目录创建一个tools文件夹，然后将tools目录加入系统PATH环境变量，供命令行使用
- 下载已经编译好的[二进制文件](https://bitbucket.org/jonforums/uru/downloads/uru-0.7.7-windows-x86.7z)，解压到tools目录中
- 然后打开命令行，执行
    
    ```
    C:\tools>uru_rt admin install
    C:\tools>uru_rt admin add system
    ```

到此安装完成。

## 增加ruby

也很简单。

- 启动命令行
- 增加包含ruby.exe的路径到uru中

    ```
    uru admin add c:\Ruby193
    uru admin add c:\Ruby21
    ```

- 使用`uru ls`看下当前安装了哪些ruby

    ```
    d:\work\open_source\super-deploy-script\ocra>uru ls
    =>  193p550     : ruby 1.9.3p550 (2014-10-27) [i386-mingw32]
        213p242     : ruby 2.1.3p242 (2014-09-19 revision 47630) [i386-mingw32]
        system      : ruby 1.8.7 (2013-06-27 patchlevel 374) [i386-mingw32]
    ```

## 切换Ruby

然后即可使用`uru 部分名称`来进行ruby的切换，比如：

```
uru 19      # 切换到ruby 1.9.3p550
---> Now using ruby 1.9.3-p550 tagged as `193p550`

uru 21      # 切换到ruby 2.1.3p242
---> Now using ruby 2.1.3-p242 tagged as `213p242`

uru 2       # 切换到ruby 2.1.3p242
---> Now using ruby 2.1.3-p242 tagged as `213p242`
```

> 更多的文档可参考uru的[wiki](https://bitbucket.org/jonforums/uru/wiki/Examples)
