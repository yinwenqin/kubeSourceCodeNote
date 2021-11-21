---

title: "Kubernetes源码-pkg-01-wait-条件定时器库"
date: 2021/11/21 22:59:53
tags:
- Kubernetes
- Golang
- 读源码


---

# Kubernetes源码-pkg-01-wait-定时(条件)轮询库

## 前言

在前面的主要组件分析过程中，有数次提及到的wait库让我记忆犹新，这是一个被高频引用的库，各个主要组件如scheduler、controller、kubelet等都常常使用wait库中的function轮询间隔(或条件)触发执行动作。整个wait库只有一个代码文件，代码行数不过400余行，本篇就来完整地分析一下这个库。

代码路径: `vendor/k8s.io/apimachinery/pkg/util/wait/wait.go`



## 分类

wait库内的各种function，大体来说都是以轮询的形式，根据时间间隔、条件判断，来确定工具执行函数是否应被继续执行。按代码中呈现，按触发形式再细化一下，各function则可以分为这几类：

| 条件类型  | 说明                                                         |
| --------- | ------------------------------------------------------------ |
| Until类   | 用得最多的类型，一般以一条chan struct{} 或context Done接收done信号作为终止轮询的依据 |
| Backoff类 | 每间隔一定的时长执行一次回溯函数，一般情况下，间隔时长随着回溯次数递增而倍数级延长，但间隔时长也会有上限值 |
| poll类    | 两条channel，一条用作传递单次执行信号用来轮询，一条用作传递done信号 |



### Until类

Untile类型有两个具体实现，分别是Until和UntilWithContext

#### Until函数

`vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:87`

```go
func Until(f func(), period time.Duration, stopCh <-chan struct{}) {
	JitterUntil(f, period, 0.0, true, stopCh)
}

```

调用的是JitterUntil函数

```go
func JitterUntil(f func(), period time.Duration, jitterFactor float64, sliding bool, stopCh <-chan struct{}) {
	var t *time.Timer
	var sawTimeout bool

	for {
    // f()执行前先判断一次stopCh是否有信号，执行完之后也还要执行一次，说明见下方
		select {
		case <-stopCh:
			return
		default:
		}

		jitteredPeriod := period
		if jitterFactor > 0.0 {
      // 如果加入抖动因子随机数，则要把间隔周期延长至原周期的随机倍数
			jitteredPeriod = Jitter(period, jitterFactor)
		}
		
    // sliding = false，则把f()的运行时间计入间隔周期内
		if !sliding {
			t = resetOrReuseTimer(t, jitteredPeriod, sawTimeout)
		}

		func() {
			defer runtime.HandleCrash()
			f()
		}()
		
    // sliding = true，则f()执行完成后继续等待间隔周期再进入下一次循环
		if sliding {
			t = resetOrReuseTimer(t, jitteredPeriod, sawTimeout)
		}

    // select下的各case分支权重相等随机分配，因此为了避免在stopCh有停止信号时却select到了t.C信号分支上，从而导致进入下一个逻辑循环超额执行了一次f()函数的情况，在代码段开头就要判断一次stopCh是否有信号
		select {
		case <-stopCh:
			return
		case <-t.C:
			sawTimeout = true
		}
	}
}
```

```go
func Jitter(duration time.Duration, maxFactor float64) time.Duration {
	if maxFactor <= 0.0 {
		maxFactor = 1.0
	}
  // 在抖动因子范围内选择随机数，放大间隔周期的倍数
	wait := duration + time.Duration(rand.Float64()*maxFactor*float64(duration))
	return wait
}

```

JitterUntil函数可谓是把条件考虑得很细致，参数上有执行周期、抖动因子、窗口期(是否包含函数执行时间)，另外在stopCh信号处理上也做到了预防超期执行，JitterUntil函数已经足以应对各类以时间间隔维度的轮询场景了。



#### UntilWithContext

`vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:96`

```go
func UntilWithContext(ctx context.Context, f func(context.Context), period time.Duration) {
	JitterUntilWithContext(ctx, f, period, 0.0, true)
}
```

```go
func JitterUntilWithContext(ctx context.Context, f func(context.Context), period time.Duration, jitterFactor float64, sliding bool) {
	JitterUntil(func() { f(ctx) }, period, jitterFactor, sliding, ctx.Done())
}
```

可以看到，最后也是调用的JitterUntil，唯一的不同只是参数stopCh换成了包了一层的context.Done()



### Backoff类

#### Backoff结构体

Backoff类函数都有一个类型为Backoff结构体的形参，先来看看这个结构体的定义

`vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:207`

```go
type Backoff struct {
	// 初始设定的间隔
	Duration time.Duration
	// 间隔时间的倍数
	Factor float64
	// 抖动因子，抖动因子是在最后计算的
	Jitter float64
  // duration最大步进次数，下面还有个专门的Step()方法，依据此字段动态计算出每次轮询的duration
	Steps int
  // 最大duration上限(计算抖动因子之前)
	Cap time.Duration
}
```

```go
func (b *Backoff) Step() time.Duration {
	if b.Steps < 1 {
    // 当Steps == 0时，duration不再变化
		if b.Jitter > 0 {
			return Jitter(b.Duration, b.Jitter)
		}
		return b.Duration
	}
  // Steps每循环一次会递减
	b.Steps--

	duration := b.Duration

	// calculate the next step
	if b.Factor != 0 {
    // duration每次轮询随着倍数因子倍数级递增
		b.Duration = time.Duration(float64(b.Duration) * b.Factor)
		if b.Cap > 0 && b.Duration > b.Cap {
      // 但duration最大也不会超过Cap
			b.Duration = b.Cap
			b.Steps = 0
		}
	}
  // 在计算的最后一步使用抖动因子放大duration，再返回最终的duration
	if b.Jitter > 0 {
		duration = Jitter(duration, b.Jitter)
	}
	return duration
}
```

ok，下面来看看具体的Backoff类实现函数。

#### ExponentialBackoff函数

```go
func ExponentialBackoff(backoff Backoff, condition ConditionFunc) error {
	for backoff.Steps > 0 {
		if ok, err := condition(); err != nil || ok {
			return err
		}
		if backoff.Steps == 1 {
			break
		}
		time.Sleep(backoff.Step())
	}
	return ErrWaitTimeout
}

```

ExponentialBackoff函数的工作逻辑是：

- 在最外层限定了backoff函数的最多重复执行次数，即等于Steps字段值

- 过程中condition()条件函数执行异常或正常则直接返回，否则按最大限定次数，每次轮询等待时间参照Step()方法返回值进行
- 到达上限次数后若条件函数仍未返回结果，则返回超时错误

显然，此函数适用于失败重试的场景，常用k8s的同学一定不会陌生，我们常遇到pod多次失败重试的状态，如:CrashLoopBackOff/ImagePullBackOff状态，重试过程正是使用此函数封装的



### poll类

#### WaitFunc结构体

```go
type WaitFunc func(done <-chan struct{}) <-chan struct{}
```

这里定义的type WaitFunc下面都会用到，接收的参数chan用作done信号传递，返回的chan用作执行信号传递



#### Poll

`vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:286`

```go
func Poll(interval, timeout time.Duration, condition ConditionFunc) error {
	return pollInternal(poller(interval, timeout), condition)
}
```

调用的pollInternal函数，看命名就知道是间隔执行的意思：

```go
func pollInternal(wait WaitFunc, condition ConditionFunc) error {
	done := make(chan struct{})
	defer close(done)
	return WaitFor(wait, condition, done)
}
```

-->

```go
func WaitFor(wait WaitFunc, fn ConditionFunc, done <-chan struct{}) error {
	stopCh := make(chan struct{})
	defer close(stopCh)
  // type WaitFunc函数返回的c是传递执行信号的chan
	c := wait(stopCh)
	for {
		select {
      // c每取出一次数据，则执行一次条件函数fn()
		case _, open := <-c:
			ok, err := fn()
			if err != nil {
				return err
			}
      // fn()条件满足，则轮询结束返回nil
			if ok {
				return nil
			}
			if !open {
				return ErrWaitTimeout
			}
		case <-done:
			return ErrWaitTimeout
		}
	}
}
```

WaitFor函数可实现按条件结果来决定对执行函数的执行次数、结束时机的控制，实现更高的可控性，但与之对应的是，执行信号、结束信号的发送全部需要在type waitFunc内部实现。即上面的Poll()函数中的`poller(interval, timeout)`是`waitFunc`的实现，来看看:

```go
func poller(interval, timeout time.Duration) WaitFunc {
	return WaitFunc(func(done <-chan struct{}) <-chan struct{} {
    // 执行信号chan
		ch := make(chan struct{})

		go func() {
			defer close(ch)
      // new一个ticker
			tick := time.NewTicker(interval)
			defer tick.Stop()
      
      // 默认无超时时间设定
			var after <-chan time.Time
			if timeout != 0 {
				// 当超时时间>0时，设定超时判定计时器
				timer := time.NewTimer(timeout)
				after = timer.C
				defer timer.Stop()
			}

			for {
				select {
				case <-tick.C:
					// 达到interval tick时间后往ch插入一个执行信号
					select {
					case ch <- struct{}{}:
					default:
					}
				case <-after:
					return
				case <-done:
					return
				}
			}
		}()

		return ch
	})
}
```

从pollInternal的实现上来看，看起来与Until差别不大，但在WaitFunc的实现层面，除了像poller()函数这样以定时器为条件外，也可以取其他更加灵活的条件作为判定往执行chan发送信号的依据。







