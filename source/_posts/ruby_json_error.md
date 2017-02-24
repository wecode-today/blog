title: ruby2.0下使用json遇到的坑
date: 2013/11/25 9:53:10
categories:
  - ruby
tags:
  - json
  - ruby
---

## ruby 2.0 json的坑

ruby2.0已经内置了对json的支持，不过在使用时遇到了一些问题。上代码：

```ruby
      File.open(omsys_file, 'r') do |f|
        omsys_data = JSON.parse(f.read)

        converter = CawGenerator::Caw::Converter.new

        ret = converter.convert(omsys_data, error_file)

        File.open(out_file, 'w') do |f|
          f.write(JSON.pretty_generate(ret))
        end
      end
```

结果代码在JSON.parse的时候抛出异常了：

```
C:/Ruby200/lib/ruby/2.0.0/json/common.rb:155:in `encode': "\xA7\x86" from GBK to UTF-8 (Encoding::UndefinedConversionError)
        from C:/Ruby200/lib/ruby/2.0.0/json/common.rb:155:in `initialize'
        from C:/Ruby200/lib/ruby/2.0.0/json/common.rb:155:in `new'
        from C:/Ruby200/lib/ruby/2.0.0/json/common.rb:155:in `parse'
```

啥原因呢？从错误来看是Json无法把GBK格式编码转换为UTF-8编码。不过我已经确认过了，要打开的json格式文件确实是UTF-8格式，怎么会是GBK编码呢？

<!-- more -->

加一行代码调试看看：

```ruby
      File.open(omsys_file, 'r') do |f|
        data = f.read
        p data.encoding
        omsys_data = JSON.parse(f.read)

        converter = CawGenerator::Caw::Converter.new

        ret = converter.convert(omsys_data, error_file)

        File.open(out_file, 'w') do |f|
          f.write(JSON.pretty_generate(ret))
        end
      end
```

输出结果 :exclamation: :

```
#<Encoding:GBK>
```

额，原来是读取的时候编码格式就错误了。ruby1.9以上版本已经支持在读取文件时指定编码格式，给读取加上UTF-8就OK了。

```ruby
      # NOTE:the omsys json file is utf-8 format.So remember to load it with r:utf-8,
      #  otherwize the load string will be GBK format.Json library can only parse utf-8
      #  data!
      #
      File.open(omsys_file, 'r:UTF-8') do |f|
        omsys_data = JSON.parse(f.read)

        converter = CawGenerator::Caw::Converter.new

        ret = converter.convert(omsys_data, error_file)

        File.open(out_file, 'w') do |f|
          f.write(JSON.pretty_generate(ret))
        end
      end
```
