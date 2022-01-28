---
title: GoStream实现BufferChan
date: 2022-01-28 17:35:25
tags: 编程
---

在[上一篇文章][1]中，我实现了Go BufferChan。但BufferChan作为一个独立库，实际使用上并不方便，所以想着把BufferChan整合进[GoStream](https://github.com/a3d21/gostream)中。

原理是，将[上一篇文章][1]的实现原理封闭在在GoStream的`BufferChan()`中。

<!-- more -->

```go
/// chan.go

// BufferChan 对channel运行缓存
// 当接收消息达到`size`或超过`timeout`未收到新消息时，发送消息
// 参数说明
//   typ  slice类型参数
//   size  缓存数量
//   timeout  超时时间
func (s Stream) BufferChan(typ interface{}, size int, timeout time.Duration) Stream {
	t := reflect.TypeOf(typ)
	if t.Kind() != reflect.Slice {
		panic("typ should be slice")
	}
	if size <= 0 {
		panic("size should gt 0")
	}
	if timeout <= 0 {
		panic("timeout should gt 0")
	}

	in := make(chan interface{})
	out := make(chan interface{})
	go s.OutChan(in)

	go func() {
		sv := reflect.MakeSlice(t, size, size)
		idx := 0

		var brush = func() {
			out <- sv.Slice(0, idx).Interface()
			sv = reflect.MakeSlice(t, size, size)
			idx = 0
		}

		for {
			select {
			case v, ok := <-in:
				if ok {
					sv.Index(idx).Set(reflect.ValueOf(v))
					idx++
					if idx == size {
						brush()
					}
				} else {
					if idx > 0 {
						brush()
					}
					close(out)
					return
				}
			case <-time.After(timeout):
				if idx > 0 {
					brush()
				}
			}
		}
	}()

	return From(out)
}

```

加点测试验证下。
```go
/// chan_test.go

func TestBufferChanBySize(t *testing.T) {
	in := make(chan int)
	go func() {
		for i := 0; i < 5; i++ {
			in <- i
		}
		close(in)
	}()

	want := [][]int{{0, 1, 2}, {3, 4}}
	got := From(in).BufferChan([]int{}, 3, time.Second).Collect(ToSlice([][]int{}))
	assert.Equal(t, want, got)
}

func TestBufferChanByTimeout(t *testing.T) {
	in := make(chan int)
	go func() {
		in <- 0
		time.Sleep(time.Millisecond * 500)
		in <- 1
		in <- 2
		time.Sleep(time.Millisecond * 500)
		in <- 3
		in <- 4
		close(in)
	}()
	want := [][]int{{0}, {1, 2}, {3, 4}}
	got := From(in).BufferChan([]int{}, 100, time.Millisecond*300).Collect(ToSlice([][]int{}))
	assert.Equal(t, want, got)
}

```

[1]: /2022/01/27/2022-01-27-go-buffer-channel/ ""