title: Ruby GIL之谜
date: 2015/04/18 21:44:00
categories:
  - ruby
tags:
  - gil
  - 并发

---
> 从[jstorimer](http://www.jstorimer.com/blogs/workingwithcode/8085491-nobody-understands-the-gil)的博客翻译

# Nobody understands the GIL Part1 
## 什么是GIL
提起Ruby的多线程，就不得不说Ruby里面的GIL。GIL全称是`Global Interpreter Lock`（全局解释器锁），那么GIL到底是啥呢？

```
MRI有一个全局的解释器锁。它用于锁住Ruby代码的执行。这意味着在一个多线程环境中，在同一个时间，只有一个线程能够执行Ruby代码。
```

因此，如果你有8个线程，工作在一个8核的CPU机器上，在任何时间中，只有一个线程能够在其中一个核上运行（不过这个不一定表示每次都运行在同一个核心上）。GIL用于保护Ruby内部，用于防止产生竞争环境(race condition)，进而导致数据损坏。

## 问题

## 往数组中增加数据不是线程安全的

在Ruby中，只有很少的操作默认是线程安全的。比如，往数组中增加元素：

```ruby
array = []

5.times.map do
  Thread.new do
    1000.times do
      array << nil
    end
  end
end.each(&:join)

puts array.size
```

这里有5个线程共享1个数组对象。每个线程往数组里面`push`1000次`nil`。所以到最后，数组里面应该有5000个nil，是吧？

```
$ ruby pushing_nil.rb
5000
```

```
$ jruby pushing_nil.rb
4446
```

```
$ rbx pushing_nil.rb
3088
```

这么一个简单地例子，已经暴露出Ruby的一个操作不是线程安全的。在这里到底发生了什么？

<!-- more -->

请注意MRI实现产生了正确的结果 5000 。JRuby和Rubinius产生了错误的结果。如果你重新运行脚本，你可能会发现MRI再一次返回正确结果，但是JRuby和Rubinius产生了不同的错误结果。    

结果不一致的原因就是应为GIL。因为MRI有GIL，虽然说有5个线程都在运行，实际上一次只有一个线程是活动的。换句话说，执行不是并行的。JRuby和Rubinius因为没有GIL，当你有5个线程在运行时，你真的有5个线程同时在所有的CPU核心上运行！

在那些支持并行的Ruby实现上，5个线程都进入运行线程不安全的代码。它们相互影响，最终导致破坏底层数据。

## 多线程如何破坏数据

怎么会这样呢？Ruby不是程序员最好的朋友吗？我会给你展示一些基于高阶解释的技术细节，也会给你展示这个从技术上来说是可能的。

当你使用MRI,JRuby,Rubinius的时候，不同的Ruby实现采用了不同的语言。MRI是C语言写的，JRuby使用Java编写，Rubinius是使用Ruby和C++混合实现的。因此当你有如下的Ruby语句：

```ruby
array << nil
```

这条语句会转换为几十上百条底层代码。以MRI的`Array#<<`方法为例：

```c
VALUE
rb_ary_push(VALUE ary, VALUE item)
{
    long idx = RARRAY_LEN(ary);

    ary_ensure_room_for_push(ary, 1);
    RARRAY_ASET(ary, idx, item);
    ARY_SET_LEN(ary, idx + 1);
    return ary;
}
```

请注意：这里至少有 **4条底层操作** 。

1. 获取当前数组的长度。
2. 检查数组中是否有空间用于插入新数据
3. 将新数据附加到数组
4. 设定数组的长度为旧长度+1

每条这种操作又会调用其他函数或者宏。我会将这个的目的是，给你看看多线程是如何损坏数据的。在单线程环境中，你可以看看这段C代码，并且可以很容易地跟踪函数的执行路径。换句话说，我们已经习惯了通过代码以线性的方式，推理整个'世界'的状态。这就是我们通常编写代码的方式。

当涉及多个线程，这不再是可能的了。当有两个线程，每个线程通过代码跟踪其自己的路径。现在，你必须保持2 （或以上）的'指针'，指向每个正在执行的线程。因为线程共享相同的内存空间（同一个进程中），两个线程可同时改变“世界”的状态。

一个线程中断另一个线程的执行，改变了另外一个线程的内部状态，然后另外的线程继续执行，完全不知道的事物的状态发生了变化。这种情况是完全可能的。

这就是为什么一些Ruby实现，往数组中简单地附加数据也会产生错误结果的原因。请看下图：

这里是我们的初始系统状态。

![append_base](http://7vzu7z.com1.z0.glb.clouddn.com/2015/4/19/append_base_grande.png)

这里有两个活动线程，同时进入了这个函数。考虑第1-4步是这个函数(`Array#<<`)实现的伪代码。一旦两个线程都进入这个函数，这里是一个可能出现的执行顺序，以线程A开始。

![append_arrows_grande](http://7vzu7z.com1.z0.glb.clouddn.com/2015/4/19/append_arrows_grande.png)

这个看起来有些复杂，但是只需要跟着箭头指向的顺序来了解这里发生了什么。在每个步骤上，我增加了小标签，用于从每个线程的角度来显示状态变化。

这只是其中一种可能的执行序列。

在此发生的是：线程A开始按顺序路径执行此函数，不过当它执行到第三步的时候，它遇到了上下文切换。因此线程A就在此地方暂停执行。此时线程B开始执行，它执行完整个函数，将元素附加到数组末尾并增加数组长度属性。

一旦线程B执行完成，线程A恢复上下文，从中断的地方继续执行。记住，线程A在增加长度属性之前被暂停，因此它继续执行并增加数组长度。只不过它并不知道线程B在它的眼皮子底下修改了状态。

线程B将长度设置为1，然后线程A将长度设置为1，两个都已经将元素附加到数组末尾。数据就这样被破坏掉了，这个事件顺序就是导致JRuby或者Rubinius结果不正确的原因。

除此之外，在JRuby和Rubinius中，通常还有更为复杂的情况并发执行。在这种情况下，一个线程被暂停后，同时并行运行的其他线程，所有的线程都有可能在同一时间处理数据。

如果你重复执行测试脚本，你会发现不正确的结果每次都不相同。这里线程切换是 **非确定性的，不可预知的** ，它完全有可能发生在函数执行前，执行后，或者没啥关系。

**那为啥Ruby不帮我们搞定这些事情呢？** 其他编程语言的数组同理也不提供线程安全保障：代价太昂贵了。其他Ruby实现要提供线程安全的数据结构是可能的，但是会导致额外的开销，最终导致单线程程序执行变慢。而开发者的责任就是在需要的时候保证线程安全。

那么问题来了，线程安全哪家强？Ruby实现找蓝翔…………

如果这样的线程切换是可能的，那为什么MRI能够输出正确的结果呢？这个线程切换又到底是啥鬼东西？(ˇˍˇ） 想～)

问题1就是我写这篇文章的原因。对于GIL的高阶理解是无法回答这个问题的。高阶理解只能知道一个时间只有一个线程在执行。但是当上下文切换在Ruby函数执行中发生的时候，会发生啥？什么是GIL语义？

但是首先……

## 都是调度器的错！

上下文切换来自于操作系统的线程调度器。在所有的Ruby实现中，一个Ruby线程对应一个操作系统原生线程。操作系统保证没有一个线程会把系统的所有资源（比如CPU时间片）全部吃完，因此操作系统实现了线程的调度器，用于在各个线程中公平分配资源。

这表现为一系列的暂停和恢复。每个线程都有机会用于占用一个时间片资源，然后它被暂停并记录上下文，其他线程就有它们的机会执行。随着时间的推移，这个线程将被恢复执行，如此反复。

这对于操作系统来说是很有效的，不过给你的程序带来了一定程度上的随机性。例如`Array#<<`方法就需要察觉它可能在任何时间被暂停，然后另外的线程并行执行同样的操作，在它的眼皮底下修改'整个世界'的状态。

**怎么办？将操作原子化。**

如果你需要确保操作不被中断，那么需要将操作原子化。如此你就确保在操作完成之前不会被中断。这样就可以防止我们前面例子的第三步，最终当它恢复执行第4步的时候，防止损坏数据。

原子化操作最简单的方式就是使用锁。以下代码确保在MRI，JRuby和Rubinius上都可以输出正确结果。

```ruby
array = []
mutex = Mutex.new

5.times.map do
  Thread.new do

    mutex.synchronize do
      1000.times do
        array << nil
      end
    end

  end
end.each(&:join)

puts array.size
```

这段代码使用了一个共享互斥量或者锁来确保执行结果正确。一旦一个线程进入了`mutex.synchronize`代码块，其他所有线程必须等到当前线程将代码块执行完成后，才能进入同样地代码。通过将操作原子化，你确保了在代码块里即使发生了上下文切换，其他线程也无法进入同样的代码。线程调度器可以看到这个，将调度重新切换到其他线程上。这同时也确保了其他线程无法修改'世界状态'。这就是线程安全。

## GIL也是一把锁

我已经给你展示了如何使用锁来将操作原子化，并提供线程安全保证。GIL也是一个锁，那么它确保了你的所有Ruby代码都是线程安全的吗？它确保`Array#<<`操作原子化吗？

# Nobody understands the GIL Part2

在第一部分我们留下了2个问题:

1. GIL确保`array << nil`是原子操作吗？
2. GIL确保你的Ruby代码线程安全吗？

第一个操作我们通过查看源代码来解决。

回忆一下上次的代码片段：

```ruby
array = []

5.times.map do
  Thread.new do
    1000.times do
      array << nil
    end
  end
end.each(&:join)

puts array.size
```

如果你假设数组是线程安全的，那么期望的结果应该是5000。但是由于数组不是线程安全的，因此JRuby和Rubinius产生了一个非预期的结果--小于5000。这就是多个线程的上下文切换导致损坏数据的原因。

**MRI产生了预期结果，但它是侥幸还是担保呢？** 我们通过查看Ruby的源代码片段来学习：

```ruby
Thread.new do
  array << nil
end
```

## 从头说起

要知道这段代码内部发生了啥，我们需要看看MRI是如何启动线程的。我们会查看MRI源代码的[thread.c](https://github.com/ruby/ruby/blob/trunk/thread.c)文件。

第一段开始的代码`Thread.new`是启动了一个原生线程来支撑ruby线程。启动线程的函数是`thread_start_func_2`。我们从大的方向来看看这个函数。

```c
static int
thread_start_func_2(rb_thread_t *th, VALUE *stack_start, VALUE *register_stack_start)
{
    int state;
    VALUE args = th->first_args;
    rb_proc_t *proc;
    rb_thread_list_t *join_list;
    rb_thread_t *main_th;
    VALUE errinfo = Qnil;
    if (th == th->vm->main_thread)
  
    ruby_thread_set_native(th);

    th->machine.stack_start = stack_start;
    thread_debug("thread start: %p\n", (void *)th);

    gvl_acquire(th->vm, th);    // **获取GIL锁**
    {
      thread_debug("thread start (get lock): %p\n", (void *)th);
      rb_thread_set_current(th);    // **设定当前线程**

      TH_PUSH_TAG(th);
      if ((state = EXEC_TAG()) == 0) {
          SAVE_ROOT_JMPBUF(th, {
                    native_set_thread_name(th);
        if (!th->first_func) {
            GetProcPtr(th->first_proc, proc);
            th->errinfo = Qnil;
            th->root_lep = rb_vm_ep_local_ep(proc->block.ep);
            th->root_svar = Qnil;
            EXEC_EVENT_HOOK(th, RUBY_EVENT_THREAD_BEGIN, th->self, 0, 0, Qundef);
            th->value = rb_vm_invoke_proc(th, proc, (int)RARRAY_LEN(args), RARRAY_CONST_PTR(args), 0);  // 执行线程代码块
            EXEC_EVENT_HOOK(th, RUBY_EVENT_THREAD_END, th->self, 0, 0, Qundef);
        }
        else {
            th->value = (*th->first_func)((void *)args);
        }
          });
      }
      else {
        ...
      }

      th->status = THREAD_KILLED;
      thread_debug("thread end: %p\n", (void *)th);

      main_th = th->vm->main_thread;
      if (main_th == th) {
          ruby_stop(0);
      }
      if (RB_TYPE_P(errinfo, T_OBJECT)) {
          /* treat with normal error object */
          rb_threadptr_raise(main_th, 1, &errinfo);
      }
      TH_POP_TAG();

      /* locking_mutex must be Qfalse */
      if (th->locking_mutex != Qfalse) {
          rb_bug("thread_start_func_2: locking_mutex must not be set (%p:%"PRIxVALUE")",
           (void *)th, th->locking_mutex);
      }

      /* delete self other than main thread from living_threads */
      rb_vm_living_threads_remove(th->vm, th);
      if (rb_thread_alone()) {
          /* I'm last thread. wake up main thread from rb_thread_terminate_all */
          rb_threadptr_interrupt(main_th);
      }

      /* wake up joining threads */
      join_list = th->join_list;
      while (join_list) {
          rb_threadptr_interrupt(join_list->th);
          switch (join_list->th->status) {
            case THREAD_STOPPED: case THREAD_STOPPED_FOREVER:
        join_list->th->status = THREAD_RUNNABLE;
            default: break;
          }
          join_list = join_list->next;
      }

      rb_threadptr_unlock_all_locking_mutexes(th);
      rb_check_deadlock(th->vm);

      if (!th->root_fiber) {
          rb_thread_recycle_stack_release(th->stack);
          th->stack = 0;
      }
    }
    native_mutex_lock(&th->vm->thread_destruct_lock);
    /* make sure vm->running_thread never point me after this point.*/
    th->vm->running_thread = NULL;
    native_mutex_unlock(&th->vm->thread_destruct_lock);
    thread_cleanup_func(th, FALSE);
    gvl_release(th->vm);  // **释放GIL**

    return 0;
}
```
 
这个函数中有很多代码，不过我们着重看加了注释的部分。在顶部，这个新线程会获取GIL。记住，这个线程在实际拿到GIL之前，都是空闲的。在代码中间，它调用你传给线程的代码块。在完成这些之后，它释放GIL并退出原生线程。

在我们的代码片段里，这个新线程是由主线程启动的。基于这个情况，我们可以假设主线程当前持有GIL。在主线程释放GIL之前，这个新线程只能等待。

我们看看当这个新线程想要获取GIL时发生了啥(Linux平台)：

```c
static void
gvl_acquire_common(rb_vm_t *vm)
{
    if (vm->gvl.acquired) {
        vm->gvl.waiting++;
        if (vm->gvl.waiting == 1) {
            /*
             * Wake up timer thread iff timer thread is slept.
             * When timer thread is polling mode, we don't want to
             * make confusing timer thread interval time.
             */
            rb_thread_wakeup_timer_thread_low();
        }

        while (vm->gvl.acquired) {
            native_cond_wait(&vm->gvl.cond, &vm->gvl.lock);
        }

        vm->gvl.waiting--;

        if (vm->gvl.need_yield) {
            vm->gvl.need_yield = 0;
            native_cond_signal(&vm->gvl.switch_cond);
        }
    }

    vm->gvl.acquired = 1;
}
```


这个是Linux平台上的`gvl_acquire_common`函数。这个函数被`gvl_acquire`函数调用，用于获取GIL。

首先它检查当前是否已经获取了GIL。如果是，那么就增加GIL的waiting属性。对于我们的执行代码来说，这个值现在应该是1。下面的代码用于检查等待值是否为1.如果是，下一行代码就触发定时器线程的唤醒。

定时器线程是MRI的秘密武器，用于MRI线程系统的顺畅，并防止任何一个线程长时间占用GIL。但在此之前，我们不要跑那么快，先让说明GIL的一些信息，再介绍该定时器线程。

![pre-gil_2_medium](http://7vzu7z.com1.z0.glb.clouddn.com/2015/4/24/pre-gil_2_medium.png)
我已经说过，MRI线程对应一个原生操作系统线程。但是这张图表示每个MRI线程是在它的原生线程上并行执行的。GIL防止了这个。我们需要将GIL加入到图中，将图标变得更为现实。

![with-gil_medium](http://7vzu7z.com1.z0.glb.clouddn.com/2015/4/24/with-gil_medium.png)

当Ruby线程想在它的原生线程里面执行代码时，它必须要首先获取GIL。**GIL作为Ruby线程和底层原生线程的中间者，严重降低了并行！在前一张图中，多个Ruby线程和其底层的原生线程都是并行执行的。第二张图更接近于MRI，在任何时间，只有一个线程能够获取到GIL，因此MRI中并行执行代码完全被禁止。

**根据MRI开发组成员的说法，GIL保护了系统内部状态。** 由于有GIL，它们不需要显示获取锁，或者对内部数据进行同步。如果两个线程无法同时修改内部状态，那么就不存在竞争环境。

对于开发者，这会严重限制你的代码在MRI上无法并行执行。

## 定时器线程

我已经说过了，定时器线程是用于防止其他线程霸占GIL的。定时器线程是MRI的一个内部原生线程，它没有关联的Ruby线程。定时器线程由MRI函数`rb_thread_create_timer_thread`方法启动。

当MRI启动后只有主线程在运行时，定时器线程处于睡眠状态。但记住，一旦一个线程开始等待GIL，它就唤醒定时器线程。

![sleeping-timer_large](http://7vzu7z.com1.z0.glb.clouddn.com/2015/4/24/sleeping-timer_large.png)

这个更接近于MRI中GIL的实现方式。右上角的线程是我们新创建的。由于它是当前唯一要获取GIL的线程，它唤醒了定时器线程。

定时器线程防止GIL被霸占。每100毫秒，定时器线程都给当前占用GIL的线程设定一个中断，通过`RUBY_VM_SET_TIMER_INTERRUPT`宏。这里的细节很重要，应为它会给我们解释`array << nil`是否是原子操作。

如果你熟悉时间片的概念，这个很类似。

每100毫秒定时器线程都会给持有GIL的线程设定一个中断标识。设定中断标志并不是一定要真正中断线程的执行。

## 处理中断标志

在`vm_eval.c`文件中，有Ruby如何调用方法的代码。它负责设定方法调用的上下文环境，并调用正确的方法。在`vm_call0_body`函数最后，在返回函数调用值之前，中断被检查。

```c
static VALUE
vm_call0_body(rb_thread_t* th, rb_call_info_t *ci, const VALUE *argv)
{
  VALUE ret;
 
  success:
    RUBY_VM_CHECK_INTS(th);
  return ret;
}
```

如果中断标志已经被设定，那么它在这个店停止执行，在返回函数调用值之前。在执行任何其他Ruby代码之前，当前线程会释放GIL，并调用`sched_yield`方法。`sched_yield`方法是系统方法，用于将线程调度器调度到其他线程上。一旦这个操作完成，被中断的线程尝试重新获取GIL，等待其他线程释放GIL。

好了，这就是我们的答案。`aray << nil`是原子的。感谢GIL，所有的C实现的Ruby方法都是原子操作。

因此这个例子：

```ruby
array = []

5.times.map do
  Thread.new do
    1000.times do
      array << nil
    end
  end
end.each(&:join)

puts array.size
```

保证每次在MRI上执行，都会产生正确的结果。

**但是记住这个保证在其他Ruby实现上是没有的。** 如果你将这个代码拿到其他没有GIL的Ruby实现上运行，就会产生一个非预期的结果。知道GIL的保证是好事，不过编写基于GIL的代码不是个好主意。如果你这样做，你的代码就只能在MRI的环境上运行了。

同样的，GIL不是一个公开的API。它没有文档也没有规格。当前有Ruby代码隐式依赖于GIL，但是MRI开发组提到以后会去掉GIL，或者改变它的语义。基于这些原因，你不应该编写基于当前GIL行为的Ruby代码。

## 非原生方法

我倒现在说的都是`array << nil`是原子的。这个很简单，因为`Array#<<`方法获取一个参数为产量值(`nil`)，而且在这个表达式里面只有一个方法调用，用C开发。即使在代码中中断，它也会继续执行到结束，并释放GIL。

那下面的代码呢？

```ruby
array << User.find(1)
```

如果`Array#<<`方法能够执行，那么它必须要先计算`User.find(1)`的值。你知道，在Rails中，`User.find(1)`在它的实现里会调用一大堆Ruby代码。

因此，用Ruby代码实现的方法就没有MRI的原子操作保证了。只有用C写的代码才有这个保证。

那么，这是否意味着`Array#<<`在上面的代码中任然是原子的？是的，但是仅限于它右边的值已经被计算过了。换句话说，`User.find(1)`方法调用没有原子性保障。它的执行结果值会作为参数传递给`Array#<<`，而这个操作有原子性保障。

## 这意味着啥？

GIL将方法调用原子化了。这对你意味着啥？

在第一部分中，我给你展示了在一段C函数中间执行时进行上下文切换的情况。有了GIL，这种情况就不可能了。如果线程切换发生了，其他的线程会保持空闲状态等待GIL，让当前的线程有机会不被中断继续执行。这个行为只在MRI中，C编写的Ruby代码才有。

这个行为消除了MRI中很多可能发生的竞争环境。从这个角度来说，GIL是MRI的一个严格内部实现。它保持了MRI的安全性。

但是还有一个问题没有得到回答。GIL是否保证了你编写的所有Ruby代码线程安全？

我们在第三篇里面会回答这个问题。