title: 拓扑映射还能再快吗？
date: 2014/12/13 11:46:25
categories:
  - ruby
tags:
  - stackprof
  - ruby
  - 优化
---

# 简介
拓扑映射是一个用Ruby实现的组网图搜索算法，用于在大的组网图中搜索满足用户要求的小组网。老版本的拓扑映射算法基于老旧的`ruby1.8.6`，用户已经多次反馈效率低下。听闻ruby推出了新的2.2版本，我们能否利用新版本ruby的高性能呢？

# 平台级优化
我们首先考虑代码不做大改动，切换平台和Ruby版本进行优化。方式如下：
- Ruby升级到最新版本 **2.2**
- 使用 **Linux** 替换Windows

我们使用一个运行时间很长的测试数据来测试性能，测试机器都是公司的E6000高配服务器，12核3.2G至强CPU(`CRuby`只能用单核，多核没意义)。测试结果请参考下图：

| Ruby版本     | 操作系统      | 计算时间（秒） |
|-------------|--------------|--------------|
|Ruby 1.8.7   |  Windows2008 |	`790`	        |
|jruby 9.0.0.0.pre1|Ubuntu 12.04.2/JAVA7|	`559`|
|Rubinius 2.5.3|Ubuntu 12.04.2|	`547`|
|jruby 9.0.0.0.pre1|Ubuntu 12.04.2/JAVA8|	`516`|
|Ruby 2.1.3|Windows2008|	`436`|
|Ruby 2.1.3|Ubuntu 12.04.2|	`324`|
|Ruby 2.2.0|Ubuntu 12.04.2|	`314`|

从数据来看，在Ubuntu平台的Ruby2.2性能最好。在执行过程中，`Rubinius`出现过随机错误，感觉不是太稳定。

<!-- more -->

# 代码级优化
通过切换操作系统和平台，通过`stackprof`分析代码性能，进行优化。参考来源见[tmm1的博客](http://tmm1.net/ruby21-profiling/)。

>注意:
>1. 以下所有优化分析/测试操作都是在Linux上执行的
>2. `stackprof`使用了Ruby2.1提供的API，因此只支持Ruby2.1以上版本


## 优化ConflictCheck
经过测试，发现某些用户开发的插件中，同/跨板约束占据了大量的时间，优化效果不佳，暂时先禁用此优化规则。

经过此次优化，时间下降到`276`秒。

## 优化SlotType约束
经过分析，发现约束中的SlotType约束占据了很多时间。

```
==================================
  Mode: cpu(1000)
  Samples: 30019 (0.06% miss rate)
  GC: 4601 (15.33%)
==================================
     TOTAL    (pct)     SAMPLES    (pct)     FRAME
     13200  (44.0%)       13200  (44.0%)     block in TopoPluginRouter::SlotAux#any?
       839   (2.8%)         839   (2.8%)     block in TestlibTopo::Topo::Restrict#validate
```

继续跟踪:

```
stackprof tmp/perf-result.dump --method 'block in TopoPluginRouter::SlotAux#any?'          
block in TopoPluginRouter::SlotAux#any? (/root/.rbenv/versions/2.2.0/lib/ruby/gems/2.2.0/gems/topo-plugin-router-2.0.5/lib/topo-plugin-router/dsl/port_dsl.rb:56)
  samples:  13200 self (44.0%)  /   13200 total (44.0%)
  callers:
    13200  (  100.0%)  TopoPluginRouter::SlotAux#any?
  code:
                                  |    56  |       items.each { |c|
                                  |    57  |         c.strip! # 去除空格，防止多余的空格引发判断问题
                                  |    58  |         c.downcase! # 都转换为小些再比较
                                  |    59  | 
                                  |    60  |         # 用户如果输入错误，则跳过
  765    (2.5%) /   765   (2.5%)  |    61  |         next if c.empty?
                                  |    62  | 
  795    (2.6%) /   795   (2.6%)  |    63  |         if c[0].chr == '!' # 如果是!表达式，则应该不包含此slottype值
                                  |    64  |           return true if port_slottype !~ /#{c.gsub(/^!/, '')}/
                                  |    65  |         else # 普通表达式则包含此slottype值
 11640   (38.8%) /  11640  (38.8%)|    66  |           return true if port_slottype =~ /#{c}/
                                  |    67  |         end
```

可以看到正则表达式匹配占据了大量的时间。

看看实现原理：

```ruby
@dtb_to_dta1.slottype("*BSU*|*LPUF-21*|*LPUF-40*|*LPUF-51*|*LPUF-101*|*LPUF-120*|*LPUF-240*|*LPUF-20*")
```

用户这样写，我们需要把具体的字符串转换为8个正则表达式，一个个进行运算，因此效率很低。其实只需要写一个正则表达式即可。我们给用户提供一个改进版本的方法`slottype`，支持传入正则表达式。

```ruby
REGEX_LPUF_BUS = /(BSU|LPUF-(21|40|51|101|120|240)|LPUF-20)/i
@dtb_to_dta1.slottype REGEX_LPUF_BUS
```

这样我们就只需要运算一次正则表达式即可，效率大为提升。

经过此次改进，时间减少为`185`秒。

## 优化top方法
缓存`top`方法，减少计算量。`top`方法被频繁调用，因此我们可以缓存返回值，减少计算。

原来：

```ruby
    def top
      if parent.nil? or parent.parent.nil?
        self
      else
        parent.top
      end
    end
```

现在：

```ruby
    def top
      @top ||= if parent.nil? or parent.parent.nil?
        self
      else
        parent.top
      end
    end
```

经过此次优化，时间减少为`149`秒。

## 优化Minienv::Env::Connect

测试数据:

```text
stackprof tmp/perf-result.dump --method 'Minienv::Env::Connect#src_port' 
Minienv::Env::Connect#src_port (/root/.rbenv/versions/2.2.0/lib/ruby/gems/2.2.0/gems/minienv-1.0.2/lib/minienv/connect.rb:20)
  samples:  3373 self (9.0%)  /   3373 total (9.0%)
  callers:
     911  (   27.0%)  Minienv::Env::Connect#cross_port
     864  (   25.6%)  Minienv::Env::Connect#half_cross?
     796  (   23.6%)  Minienv::Env::Connect#uncross_port
```

可见`cross_port`,`half_cross?`,`uncross_port`访问次数比较多，可以利用缓存减少访问次数。优化原理同上。

源代码：
```ruby
      def cross_port
        if src_port.top.physical_switch?
          src_port
        else
          dst_port
        end
      end
```

修改后:
```ruby
      def cross_port
        @cross_port ||= if src_port.top.physical_switch?
          src_port
        else
          dst_port
        end
      end
```

经过此次优化，时间减少为`131`秒。

## 优化TestlibTopo::Match::ConnectorMatcher

```text
==================================
  Mode: cpu(1000)
  Samples: 32968 (0.05% miss rate)
  GC: 3441 (10.44%)
==================================
     TOTAL    (pct)     SAMPLES    (pct)     FRAME
      3402  (10.3%)        3365  (10.2%)     rescue in TestlibTopo::Match::ConnectorMatcher#recheck_when_loop
```

跟踪方法输出：
```text
rescue in TestlibTopo::Match::ConnectorMatcher#recheck_when_loop (/root/.rbenv/versions/2.2.0/lib/ruby/gems/2.2.0/gems/hunter-algorithm-2.0.0/lib/hunter/match/rule/connector_matcher.rb:64)
  samples:  3365 self (10.2%)  /   3402 total (10.3%)
  callers:
    3402  (  100.0%)  TestlibTopo::Match::ConnectorMatcher#recheck_when_loop
  callees (37 total):
      17  (   45.9%)  Minienv::Envbase#top
      17  (   45.9%)  Minienv::Env::Connect#src_port
       3  (    8.1%)  Minienv::Env::Connect#dst_port
  code:
                                  |    64  |       yield
                                  |    65  | 
  284    (0.9%) /   284   (0.9%)  |    66  |     rescue TestlibTopo::MatchException
                                  |    67  |       # 如果直连链接是自环的，那么要反方向再尝试一次
   39    (0.1%) /     2   (0.0%)  |    68  |       if env_link.src_port.top == env_link.dst_port.top
                                  |    69  |         send method_name, env_link, candidate
                                  |    70  |       else
 3079    (9.3%) /  3079   (9.3%)  |    71  |         raise
                                  |    72  |       end
```

可以看到是`raise`方法占用了大量时间。通过分析代码，发现里面是用了异常来控制执行流程。

```ruby
    rescue TestlibTopo::MatchException
      # 如果直连链接是自环的，那么要反方向再尝试一次
      if env_link.src_port.top == env_link.dst_port.top
        send method_name, env_link, candidate
      else
        raise
      end

    end
```

用异常控制执行流程代价很高，因此我们将其修改为普通的`if else`判断即可。

```ruby
      if yield
        true
      else
        if env_link.src_port.top == env_link.dst_port.top
          send method_name, env_link, candidate
        else
          false
        end
      end
```

经过此次优化，时间下降到`109`秒。

## 优化Minienv::Envbase#objname

继续采样：
```text
==================================
  Mode: cpu(1000)
  Samples: 27500 (0.07% miss rate)
  GC: 2539 (9.23%)
==================================
     TOTAL    (pct)     SAMPLES    (pct)     FRAME
      2780  (10.1%)        2780  (10.1%)     Minienv::Envbase#objname
```

跟踪方法：
```text
Minienv::Envbase#to_s (/root/.rbenv/versions/2.2.0/lib/ruby/gems/2.2.0/gems/minienv-1.0.3/lib/minienv/env_base.rb:37)
  samples:    30 self (0.1%)  /   2676 total (9.7%)
  callers:
    1401  (   52.4%)  block (2 levels) in <module:TopoPluginRouter>
     413  (   15.4%)  block (2 levels) in TestlibTopo::Selector::CrossLinkSelector#each
```

继续跟踪：
```text
  callees (9787 total):
    3601  (   36.8%)  Log4r::Logger.[]
    2090  (   21.4%)  #<Log4r::Logger:0x007f38311ac460>.debug
```

我顶，竟然跟踪到日志里面去了。我们之前使用`log4r`的配置文件中，不是已经把日志关闭了吗？看看代码：

```ruby
  def self.debug(str = nil)
    log = Log4r::Logger['toporesultlogger']
    if log.debug?
      if block_given?
        log.debug(yield)
      else
        log.debug(str)
      end
    end
  end
```

原来log没有缓存，导致每次都访问。还有全局日志设置为`DEBUG`，导致`log.debug(yield)`这一句仍然要执行，完全无必要，修改为`WARN`即可不执行`DEBUG`语句。

因此，修改为：

```ruby
  @@log = Log4r::Logger['toporesultlogger']
  def self.debug(str = nil)
    if @@log.debug?
      if block_given?
        @@log.debug(yield)
      else
        @@log.debug(str)
      end
    end
  end
```

log4r日志配置文件：

```yaml
log4r_config:
  pre_config:
    global:
      level: WARN
```

优化完成后，执行时间为`63`秒。

## 优化Minienv::Envbase#parent
继续分析，执行采样：

```text
==================================
  Mode: cpu(1000)
  Samples: 15961 (0.11% miss rate)
  GC: 1326 (8.31%)
==================================
     TOTAL    (pct)     SAMPLES    (pct)     FRAME
      1012   (6.3%)        1012   (6.3%)     Minienv::Envbase#parent
      2064  (12.9%)         946   (5.9%)     Minienv::Envbase#descendants
```

分析parent方法：
```text
Minienv::Envbase#parent (/root/.rbenv/versions/2.2.0/lib/ruby/gems/2.2.0/gems/minienv-1.0.3/lib/minienv/env_base.rb:49)
  samples:  1012 self (6.3%)  /   1012 total (6.3%)
  callers:
     413  (   40.8%)  Minienv::Envbase#ancestors
     282  (   27.9%)  block (2 levels) in <module:TopoPluginRouter>
     117  (   11.6%)  block in TopoPluginRouter::PortDsl#slottype!
```

可见`Minienv::Envbase#ancestors`耗用时间比较多，应该考虑使用缓存。

优化后执行时间为`59`秒。

## 优化Minienv::Envbase#descendants

继续分析：

```text
==================================
  Mode: cpu(1000)
  Samples: 14920 (0.12% miss rate)
  GC: 1245 (8.34%)
==================================
     TOTAL    (pct)     SAMPLES    (pct)     FRAME
      1988  (13.3%)         890   (6.0%)     Minienv::Envbase#descendants
```

该来的还是会来的……继续做方法缓存。需要注意的是：如果用户的插件不修改组网图，那么方法缓存的数据是没有问题的。如果用户插件修改修改了组网图结构，那么可能会有问题。

## 优化剩余方法
经过分析，基本上都是要缓存方法，就不再详细说明了。

## 最终结果

```text
==================================
  Mode: cpu(1000)
  Samples: 11537 (0.16% miss rate)
  GC: 971 (8.42%)
==================================
     TOTAL    (pct)     SAMPLES    (pct)     FRAME
       704   (6.1%)         704   (6.1%)     Minienv::Env::Connect#src_port
       623   (5.4%)         623   (5.4%)     block in TestlibTopo::Topo::Restrict#validate
```

最终经过分析，采样的代码基本上都是在执行约束了，没有进一步优化空间。下一步优化只能从算法本身来优化：

- 增加优化规则，过滤一些分支
- 增加多线程并发运算（`CRuby`暂时不支持，考虑使用`JRuby`或者`Rubinius`）

最终优化效果：

| Ruby版本     | 操作系统      | 计算时间（秒） |      优化步骤      |
|-------------|--------------|--------------|------------------|
|Ruby 1.8.7   | Windows2008  |	`790`	    |                  |   
|jruby 9.0.0.0.pre1|Ubuntu 12.04.2/JAVA7|	`559`|	替换为jruby|
|Rubinius 2.5.3|Ubuntu 12.04.2|	`547`       |替换为Rbx          |
|jruby 9.0.0.0.pre1|Ubuntu 12.04.2/JAVA8	|`516`|	升级Java版本|  
|Ruby 2.1.3   |Windows2008   |  `436`       | 升级CRuby到`2.1.3`|
|Ruby 2.1.3   |Ubuntu 12.04.2|	`324`       |	替换操作系统为Ubuntu|
|Ruby 2.2.0   |Ubuntu 12.04.2|	`314`	    | 升级CRuby到`2.2.0`|
|Ruby 2.2.0   |Ubuntu 12.04.2|	`276`	    |去掉ConflictCheck检查|
|Ruby 2.2.0   |Ubuntu 12.04.2|  `185`       |优化SlotType约束|
|Ruby 2.2.0   |Ubuntu 12.04.2|  `149`       |优化top方法|
|Ruby 2.2.0   |Ubuntu 12.04.2|  `131`       |优化uncross_port方法|
|Ruby 2.2.0   |Ubuntu 12.04.2|  `109`       |优化ConnectorMatcher类|
|Ruby 2.2.0   |Ubuntu 12.04.2|  `63`       |优化日志Envbase#objname|
|Ruby 2.2.0   |Ubuntu 12.04.2|  `46`       |最终优化|

# 结论
总结了一下算法优化的方法：

1. 使用高版本Ruby，一般新版本Ruby在性能方面比老版本都要高
2. 尽量使用Linux平台，效率上会比Windows平台高
3. 使用性能分析器(Profiler)对算法进行分析，找到消耗CPU最多的代码进行优化
    - 去掉不需要的计算代码
    - 缓存计算结果，避免重复计算
    - 不要使用异常控制执行流程
    - 能合并的操作尽量合并
    - 注意日志记录对性能的影响
