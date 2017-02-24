title: 使用SimpleCov进行Ruby代码覆盖率统计
date: 2014/10/24 17:32:10
categories:
  - ruby
tags:
  - 覆盖率
---
还在为Ruby代码的质量担心吗？`SimpleCov`来帮助你了！

## SimpleCov是啥

`SimpleCov`是一个Ruby代码覆盖率分析工具。它使用Ruby内置的代码覆盖率库，但是提供一个更清晰、简单的API用于过滤、分组、合并、格式化和显示这些数据。

看看在我们项目中实际应用中的效果：

<!-- more -->

代码覆盖率总览：

![cov1](http://7vzu7z.com1.z0.glb.clouddn.com/2014-10-27_simplecov_group.png)

单个文件代码覆盖率：

![cov2](http://7vzu7z.com1.z0.glb.clouddn.com/2014-10-27_simplecov_file.png)

## 如何使用

`SimpleCov`只支持Ruby1.9以上版本（1.8请使用rcov），以`minitest`举例说明:

1. 首先在`Gemfile`中增加`SimpleCov`

    ```ruby
    gem 'simplecov', :require => false, :group => :test
    ```

2. 然后在rakefile中增加

    ```ruby
    desc 'Run test with simple coverage test'
    task :cov do
      require 'simplecov'
      SimpleCov.start

      Dir.glob(File.join(__dir__, 'test', '**', '*.rb')) { |file|
        require file
      }
    end
    ```

    **注意:`SimpleCov.start`必须在你的测试代码之前运行，才能跟踪代码覆盖率。之后加入没有任何效果！**

3. 运行rake任务

    ```
    rake cov
    ```

4. 在`coverage`目录查看覆盖率报告

总体来说，`simplecov`是一个简单易用的代码率覆盖统计工具，值得一试。

> 更详细的文档资料请参考：https://github.com/colszowka/simplecov
