---
title: 实现GoChannel缓存——BufferChan
date: 2022-01-27 15:47:50
tags: 编程
---

channel 是一种消息通信方式，常用于异步通信。

在通信过程中，将多个消息按一定**数量**或**时间间隔**缓存起来再批量发送，是一种常见的优化方式。常见的策略是，当消息数达到`size`或超时`timeout`未收到消息时触发一次消息。

Go实现如下。

<!-- more -->

```go
/// bufferchan.go

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

		var flush = func() {
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
						flush()
					}
				} else {
					if idx > 0 {
						flush()
					}
					close(out)
					return
				}
			case <-time.After(timeout):
				if idx > 0 {
					flush()
				}
			}
		}
	}()

	return From(out)
}


```

加一些测试。

```go
/// bufferchan_test.go


func TestSizeBuffer(t *testing.T) {
	ctx := context.Background()

	in := make(chan interface{})
	go func() {
		for i := 0; i < 10; i++ {
			in <- i
		}
	}()

	out := BufferChan(ctx, in, BufferConfig{
		Size:    func() int { return 3 },
		Timeout: func() time.Duration { return time.Second },
	})

	want := []interface{}{
		[]interface{}{0, 1, 2},
		[]interface{}{3, 4, 5},
		[]interface{}{6, 7, 8},
		[]interface{}{9}}
	var got []interface{}
	for i := 0; i < 4; i++ {
		got = append(got, <-out)
	}

	if !reflect.DeepEqual(got, want) {
		t.Errorf("%v != %v", got, want)
	}
}

func TestTimeoutBuffer(t *testing.T) {
	ctx := context.Background()

	in := make(chan interface{})
	go func() {
		in <- 0
		time.Sleep(time.Millisecond * 500)
		in <- 1
		in <- 2
		time.Sleep(time.Millisecond * 500)
		in <- 3
		in <- 4
		in <- 5
	}()

	out := BufferChan(ctx, in, BufferConfig{
		Size:    func() int { return 100 },
		Timeout: func() time.Duration { return time.Millisecond * 300 },
	})

	want := []interface{}{
		[]interface{}{0},
		[]interface{}{1, 2},
		[]interface{}{3, 4, 5},
	}
	var got []interface{}
	for i := 0; i < 3; i++ {
		got = append(got, <-out)
	}

	if !reflect.DeepEqual(got, want) {
		t.Errorf("%v != %v", got, want)
	}
}

```