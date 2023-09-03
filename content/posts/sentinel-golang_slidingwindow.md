+++ 
draft = false
date = 2023-09-03T10:06:39+08:00
title = "学习 sentinel-golang 的滑动窗口实现"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++
## 前言
最近工作中需要用到限频的功能，限制同IP在一定时间内的访问量，又因为想做成统计周期可配置，所以令牌桶不太适用，需要用滑动窗口来实现，发现了一个专门做流量访问控制的阿里的开源项目[**sentinel**](link:https://github.com/alibaba/sentinel-golang)，奈何自己太菜了，直接引用了其中的滑动窗口的实现，不过还是学习了人家的写法并记录在这里


## 滑动窗口的基本属性
滑动窗口有两个最基本的属性，长度以及分桶数；长度决定了这个窗口有多长，也就是统计周期；分桶数决定了这个窗口的统计颗粒度。

那么滑动窗口该怎么实现呢？

滑动窗口不断向前滚动，采用新的数据，抛弃旧的数据，就很容易让人联想到环形数组。都是不断淘汰旧的数据，每次关心的只有这个环的大小，更新的和更旧的数据都不关心。

一起看一下sentinel-golang中的滑动窗口实现吧

## sentinel-golang的滑动窗口实现
### 定义一个桶
首先定义一个桶，最简单的桶就是一个counter，下面的`MetricBucket`就定义了一个桶，这里的counter是二维统计，可以将Pass, Reject, Total等维度分别统计，下面的`minRT`和`maxConcurrency`可以暂时不用关心， 可以看出一个桶本质上就是一个counter
```Go
type MetricBucket struct {
	// Value of statistic
	counter        [base.MetricEventTotal]int64
	minRt          int64
	maxConcurrency int32
}
```
通过面的函数可以对counter值做加法
```Go
func (mb *MetricBucket) addCount(event base.MetricEvent, count int64) {
	atomic.AddInt64(&mb.counter[event], count)
}
```
上面的桶放之四海皆可，但是如果想成为个滑动窗口用桶，只有counter是不够的，因为滑动窗口中的每一个桶还有时间的概念，BucketWrap是对桶的封装，添加了开始时间`BucketStart`，Value中存放的是一个`*MetricBucket`
```
type BucketWrap struct {
	// BucketStart represents start timestamp of this statistic bucket wrapper.
	BucketStart uint64
	// Value represents the actual data structure of the metrics (e.g. MetricBucket).
	Value atomic.Value
}
```

### 定义桶数组
`AtomicBucketWrapArray`是一个环形数组的实现
``` Go
// AtomicBucketWrapArray represents a thread-safe circular array.
//
// The length of the array should be provided on-create and cannot be modified.
type AtomicBucketWrapArray struct {
	// The base address for real data array
	base unsafe.Pointer
	// The length of slice(array), it can not be modified.
	length int
	data   []*BucketWrap
}
```

`LeapArray`则是在环形数组的基础上加入了滑动窗口的三个属性，桶长度、分桶数、滑动窗口总长度，使环形数组成为一个滑动窗口，这个是滑动窗口的核心结构
```Go
// LeapArray represents the fundamental implementation of a sliding window data-structure.
//
// Some important attributes: the sampleCount represents the number of buckets,
// while intervalInMs represents the total time span of the sliding window.
//
// For example, assuming sampleCount=5, intervalInMs is 1000ms, so the bucketLength is 200ms.
// Let's give a diagram to illustrate.
// Suppose current timestamp is 1188, bucketLength is 200ms, intervalInMs is 1000ms, then
// time span of current bucket is [1000, 1200). The representation of the underlying structure:
//
//	 B0       B1      B2     B3      B4
//	 |_______|_______|_______|_______|_______|
//	1000    1200    400     600     800    (1000) ms
//	       ^
//	    time=1188
type LeapArray struct {
	bucketLengthInMs uint32
	// sampleCount represents the number of BucketWrap.
	sampleCount uint32
	// intervalInMs represents the total time span of the sliding window (in milliseconds).
	intervalInMs uint32
	// array represents the internal circular array.
	array *AtomicBucketWrapArray
	// updateLock is the internal lock for update operations.
	updateLock mutex
}
```

### 关键函数
对于滑动窗口来说，最重要的两件事情，一个是计数器+1，一个是获得窗口内所有分桶的统计和，下面分别介绍

#### 计数器+1
计数器+1可以分解成下面两件事情  
- 根据当前时间找到对应的桶
- 对这个桶的计数器+1
下面的函数也清楚的描述了这两个步骤
```Go
func (bla *BucketLeapArray) addCountWithTime(now uint64, event base.MetricEvent, count int64) {
	b := bla.currentBucketWithTime(now) // 根据当前时间找到对应的桶
	if b == nil {
		return
	}
	b.Add(event, count) // 对这个桶的计数器做加法
}
```

根据当前时间找到对应的桶
```Go
func (la *LeapArray) currentBucketOfTime(now uint64, bg BucketGenerator) (*BucketWrap, error) {
	if now <= 0 {
		return nil, errors.New("Current time is less than 0.")
	}

	idx := la.calculateTimeIdx(now)
	bucketStart := calculateStartTime(now, la.bucketLengthInMs)

	for { //spin to get the current BucketWrap
		old := la.array.get(idx)
		if old == nil {
			// because la.array.data had initiated when new la.array
			// theoretically, here is not reachable
			newWrap := &BucketWrap{
				BucketStart: bucketStart,
				Value:       atomic.Value{},
			}
			newWrap.Value.Store(bg.NewEmptyBucket())
			if la.array.compareAndSet(idx, nil, newWrap) {
				return newWrap, nil
			} else {
				runtime.Gosched()
			}
		} else if bucketStart == atomic.LoadUint64(&old.BucketStart) {
			return old, nil
		} else if bucketStart > atomic.LoadUint64(&old.BucketStart) {
			// current time has been next cycle of LeapArray and LeapArray dont't count in last cycle.
			// reset BucketWrap
			if la.updateLock.TryLock() {
				old = bg.ResetBucketTo(old, bucketStart)
				la.updateLock.Unlock()
				return old, nil
			} else {
				runtime.Gosched()
			}
		} else if bucketStart < atomic.LoadUint64(&old.BucketStart) {
			if la.sampleCount == 1 {
				// if sampleCount==1 in leap array, in concurrency scenario, this case is possible
				return old, nil
			}
			// TODO: reserve for some special case (e.g. when occupying "future" buckets).
			return nil, errors.New(fmt.Sprintf("Provided time timeMillis=%d is already behind old.BucketStart=%d.", bucketStart, old.BucketStart))
		}
	}
}
```

先是通过下面两个函数确定桶在环形数组的idx，以及这个桶的起始时间应该是多少。

根据当前时间，算出对应的桶，在环形数组的那一个位置，这个是环形数组的经典的取模运算
```Go
func (la *LeapArray) calculateTimeIdx(now uint64) int {
	timeId := now / uint64(la.bucketLengthInMs)
	return int(timeId) % la.array.length
}

```

根据当前时间，算出对应分桶的开始时间是多少
```Go
func calculateStartTime(now uint64, bucketLengthInMs uint32) uint64 {
	return now - (now % uint64(bucketLengthInMs))
}
```

这里为什么要算起始时间呢？  
因为环形数组上当前idx所在的桶，他不一定就是正确的桶，可能需要更新，比如在环形窗口最新和最旧桶的交接处，滑动窗口需要向前滑动一个桶的位置了，那么就需要淘汰最旧的桶，并将其数据清零。

接着是一个for循环

```Go
	for { //spin to get the current BucketWrap
		old := la.array.get(idx)
		if old == nil {
			// because la.array.data had initiated when new la.array
			// theoretically, here is not reachable
			newWrap := &BucketWrap{
				BucketStart: bucketStart,
				Value:       atomic.Value{},
			}
			newWrap.Value.Store(bg.NewEmptyBucket())
			if la.array.compareAndSet(idx, nil, newWrap) {
				return newWrap, nil
			} else {
				runtime.Gosched()
			}
		} else if bucketStart == atomic.LoadUint64(&old.BucketStart) {
			return old, nil
		} else if bucketStart > atomic.LoadUint64(&old.BucketStart) {
			// current time has been next cycle of LeapArray and LeapArray dont't count in last cycle.
			// reset BucketWrap
			if la.updateLock.TryLock() {
				old = bg.ResetBucketTo(old, bucketStart)
				la.updateLock.Unlock()
				return old, nil
			} else {
				runtime.Gosched()
			}
		} else if bucketStart < atomic.LoadUint64(&old.BucketStart) {
			if la.sampleCount == 1 {
				// if sampleCount==1 in leap array, in concurrency scenario, this case is possible
				return old, nil
			}
			// TODO: reserve for some special case (e.g. when occupying "future" buckets).
			return nil, errors.New(fmt.Sprintf("Provided time timeMillis=%d is already behind old.BucketStart=%d.", bucketStart, old.BucketStart))
		}
	}
```
在for循环中，比较真实的startTime和环形数组上桶的startTime，有三种情况

- 如果桶的startTime小了，就说明这个桶需要更新，调用`ResetBucketTo`对桶的startTime做更新，以及Reset所有数据；  
- 如果相等，那么直接返回这个桶；  
- 如果桶的startTime反而大，那么只有在并发情况下，且整个窗口只有一个分桶的时候才可能出现  

只有一个分桶的时候才可能出现startTime大于实际值的情况呢？  
因为只有一个分桶的时候，无论怎么算idx，都是会返回idx=0，
假如starTime是t0，有前后两个时间，分别比t0稍大和稍小，并发下，比t0稍大的那个goroutine更新了这个桶，startTime变更了startTime+bucketLen，那么这时候比t0稍小的goroutine运行到这里就会到这个分枝

如果有至少两个分桶，那么上述的分别比t0稍大和稍小的两个请求一定会算到不同的桶上


注意在更新桶的时候用了这样的结构，这样可以在并发的时候，发现别的goroutine正在更新桶，那么就先让别人更新，让出线程，去调度到别的goroutine，这样自己继续for循环的时候就可以走到startTime相等的分支，直接返回即可了，也不会占用goroutine，等待锁的返回
```Go
if la.updateLock.TryLock() {
    old = bg.ResetBucketTo(old, bucketStart)
    la.updateLock.Unlock()
    return old, nil
} else {
    runtime.Gosched()
}
```

找到对应桶后，对桶计数器+1，本质上是这样的函数
```Go
func (mb *MetricBucket) addCount(event base.MetricEvent, count int64) {
	atomic.AddInt64(&mb.counter[event], count)
}
```


#### 获得窗口内所有分桶的统计和
就像下面函数描述的那样，这个也分为两个步骤
- 根据当前时间获取在有效时间范围内的所有桶
- 把这些桶的计数加起来  
```Go
func (m *SlidingWindowMetric) getSumWithTime(now uint64, event base.MetricEvent) int64 {
	satisfiedBuckets := m.getSatisfiedBuckets(now) // 根据当前时间获取在有效时间范围内的所有桶
	return m.count(event, satisfiedBuckets) // 把这些桶的计数加起来
}
```

第一步获取在有效时间范围内的所有桶

1. 首先什么是有效时间范围
```Go
// getBucketStartRange returns start time range of the bucket for the provided time.
// The actual time span is: [start, end + in.bucketTimeLength)
func (m *SlidingWindowMetric) getBucketStartRange(timeMs uint64) (start, end uint64) {
	curBucketStartTime := calculateStartTime(timeMs, m.real.BucketLengthInMs())
	end = curBucketStartTime
	start = end - uint64(m.intervalInMs) + uint64(m.real.BucketLengthInMs())
	return
}
```
2. 然后获取这个区间内的所有桶
通过下面的三层调用，把判断是否有效的方法传给下层，然后在`ValuesConditional`函数中遍历所有桶判断是否满足条件，非空、未过期且有效
```Go
satisfiedBuckets := m.real.ValuesConditional(now, func(ws uint64) bool {
    return ws >= start && ws <= end
})

func (bla *BucketLeapArray) ValuesConditional(now uint64, predicate base.TimePredicate) []*BucketWrap {
	return bla.data.ValuesConditional(now, predicate)
}

// ValuesConditional returns all buckets of which the startTimestamp satisfies the given timestamp condition (predicate).
// The function uses the parameter "now" as the target timestamp.
func (la *LeapArray) ValuesConditional(now uint64, predicate base.TimePredicate) []*BucketWrap {
	if now <= 0 {
		return make([]*BucketWrap, 0)
	}
	ret := make([]*BucketWrap, 0, la.array.length)
	for i := 0; i < la.array.length; i++ {
		ww := la.array.get(i)
		if ww == nil || la.isBucketDeprecated(now, ww) || !predicate(atomic.LoadUint64(&ww.BucketStart)) {
			continue
		}
		ret = append(ret, ww)
	}
	return ret
}

其中isBucketDeprecated定义如下，即当前时间和桶的startTime的时间差不超过滑动窗口的长度
// isBucketDeprecated checks whether the BucketWrap is expired, according to given timestamp.
func (la *LeapArray) isBucketDeprecated(now uint64, ww *BucketWrap) bool {
	ws := atomic.LoadUint64(&ww.BucketStart)
	return (now - ws) > uint64(la.intervalInMs)
}
```

其实在这里case里predicate的条件已经能保证桶不过期了，但是这样写是为了通用吧，因为predicate可以传入任何方法

获取所有桶之后，对他们的数据相加得到结果
```Go
func (m *SlidingWindowMetric) count(event base.MetricEvent, values []*BucketWrap) int64 {
	ret := int64(0)
	for _, ww := range values {
		mb := ww.Value.Load()
		if mb == nil {
			logging.Error(errors.New("nil BucketWrap"), "Current bucket value is nil in SlidingWindowMetric.count()")
			continue
		}
		counter, ok := mb.(*MetricBucket)
		if !ok {
			logging.Error(errors.New("type assert failed"), "Fail to do type assert in SlidingWindowMetric.count()", "expectType", "*MetricBucket", "actualType", reflect.TypeOf(mb).Name())
			continue
		}
		ret += counter.Get(event)
	}
	return ret
}
```

## 其他收获
### `TryLock`和`Gosched`的组合用法
检测到并发时，让其他goroutine获得调度，如果不是在这学到我可能自己怎么也想不到这样写。。。
```Go
for {
    if la.updateLock.TryLock() {
    old = bg.ResetBucketTo(old, bucketStart)
    la.updateLock.Unlock()
    return old, nil
    } else {
        runtime.Gosched()
    }
    if ohter {
        break
    }
}

/////////
// tryLock的定义
const mutexLocked = 1 << iota

// The mutex which supports try-locking.
type mutex struct {
	sync.Mutex
}

// TryLock acquires the lock only if it is free at the time of invocation.
func (tl *mutex) TryLock() bool {
	return atomic.CompareAndSwapInt32((*int32)(unsafe.Pointer(&tl.Mutex)), 0, mutexLocked)
}
```
### unsafe.Pointer用法
在初始化环形数组的时候用到了`*util.SliceHeader`
```Go
sliHeader := (*util.SliceHeader)(unsafe.Pointer(&ret.data))
ret.base = unsafe.Pointer((**BucketWrap)(unsafe.Pointer(sliHeader.Data)))

/////
// SliceHeader 的定义
// SliceHeader is a safe version of SliceHeader used within this project.
type SliceHeader struct {
	Data unsafe.Pointer
	Len  int
	Cap  int
}
``` 
不知道为什么不可以直接这样
```
ret.base = unsafe.Pointer(sliHeader.Data))
```
这个问题在repo进行了提问。。 请戳 https://github.com/alibaba/sentinel-golang/issues/546  
以及对于第二行`(**BucketWrap)(unsafe.Pointer)`的转换也不是很懂


## 总结
通过环形数组实现滑动窗口，记住滑动窗口最终要的两个属性，长度和分桶数，它们决定了统计周期、粒度、滑动步长。以及最重要的两个方法：`AddCount`和`GetSum`。代码中关于`unsafe.Pointer`还有一些没彻底读懂的地方，下次继续学习吧！