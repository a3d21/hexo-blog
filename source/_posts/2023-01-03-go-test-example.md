---
title: Go测试示例
date: 2023-01-03 17:57:57
tags: [编程, 测试]
---

这篇文章的目的，是分类汇总Go测试示例。当你准备编写代码测试时，可以作为参考。

<!-- more -->

## 工具方法测试
目的：测试工具包、静态方法功能和边界条件。
如何编写： 使用[gotests](https://github.com/cweill/gotests)，或者IDE自动生成

```go
// PartitionBy 对slice按size分区
func PartitionBy(vs interface{}, size int) interface{} {
	if size < 1 {
		panic("illegal size")
	}
	t := reflect.TypeOf(vs)
	if t.Kind() != reflect.Slice {
		panic("typ should be slice")
	}

	v := reflect.ValueOf(vs)
	vlen := v.Len()
	length := (vlen + size - 1) / size

	resultValue := reflect.MakeSlice(reflect.SliceOf(t), length, length)

	for i := 0; i < length; i++ {
		begin := i * size
		end := begin + size
		if end > vlen {
			end = vlen
		}
		resultValue.Index(i).Set(v.Slice(begin, end))
	}

	return resultValue.Interface()
}

func TestPartitionBy(t *testing.T) {
	type args struct {
		vs   interface{}
		size int
	}
	tests := []struct {
		name string
		args args
		want interface{}
	}{
		{"empty slice", args{[]int{}, 1}, [][]int{}},
		{"by 1", args{[]int{1, 2, 3}, 1}, [][]int{{1}, {2}, {3}}},
		{"by 2", args{[]int{1, 2, 3}, 2}, [][]int{{1, 2}, {3}}},
		{"by 3", args{[]int{1, 2, 3}, 3}, [][]int{{1, 2, 3}}},
		{"by 4", args{[]int{1, 2, 3}, 4}, [][]int{{1, 2, 3}}},
		{"by 5", args{[]int{1, 2, 3}, 5}, [][]int{{1, 2, 3}}},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := PartitionBy(tt.args.vs, tt.args.size); !reflect.DeepEqual(got, tt.want) {
				t.Errorf("PartitionBy() = %v, want %v", got, tt.want)
			}
		})
	}
}

```

## 属性测试
属性测试源于**函数式编程**，是一种更严格的工具包/静态方法测试，用于断言方法的某种与参数无关的属性约束。
Go属性测试可以参考： https://learnku.com/docs/learn-go-with-tests/intro-to-property-based-tests/6145


```go
func TestBase64EncodeDecodeSpec(t *testing.T) {
   //base64加解密后，数据相同
   assertion := func(src []byte) bool {
      s := base64.StdEncoding.EncodeToString(src)
      got, err := base64.StdEncoding.DecodeString(s)
      if err != nil {
         return false
      }

      return reflect.DeepEqual(src, got)
   }

   if err := quick.Check(assertion, &quick.Config{
      MaxCount: 2000,
   }); err != nil {
      t.Error(err)
   }
}
```

## Suite Test
一个 Suite由Setup（初始化）、Test（测试）、TearDown（清理）组成，适用于测试对环境有依赖的组件，比如RedisClient、DB ORM等。

```go

type RedisClientTestSuite struct {
	suite.Suite

	client *redis.Client
	ctx    context.Context
}

func (s *RedisClientTestSuite) SetupSuite() {
	addr := "localhost:6379"
	if t := os.Getenv("REDIS_ADDR"); t != "" {
		addr = t
	}
	s.T().Logf("load redis addr: %s", addr)

	rdb := redis.NewClient(&redis.Options{
		Addr:     addr,
		Password: "", // no password set
		DB:       0,  // use default DB
	})
	ctx := context.Background()
	_, err := rdb.Ping(ctx).Result()
	assert.Nil(s.T(), err)

	s.client = rdb
	s.ctx = ctx
}

func (s *RedisClientTestSuite) TestSetAndGet() {
	key := "xDesTzVCDcN6DCwt"
	val := "NVGX8fmJJvZt2snR"
	_, err := s.client.Set(s.ctx, key, val, time.Second*5).Result()
	assert.Nil(s.T(), err)

	got, err := s.client.Get(s.ctx, key).Result()
	assert.Nil(s.T(), err)
	assert.Nil(s.T(), err)
	assert.Equal(s.T(), val, got)
}

func (s *RedisClientTestSuite) TestGetNotExistKeyShouldNilErr() {
	_, err := s.client.Get(s.ctx, "NOT_EXIST").Result()
	assert.NotNil(s.T(), err)
	assert.True(s.T(), errors.Is(err, redis.Nil))
}

func TestRedisClientSuite(t *testing.T) {
	suite.Run(t, new(RedisClientTestSuite))
}
```

## 服务测试
服务测试验证服务业务逻辑。服务测试与环境实现无关，可以在不同环境运行。
每个服务测试对应一个用户用例，采用 gherkin语法（given-when-then）定义。
可用框架：[GoConvey](https://github.com/smartystreets/goconvey/)、[ginkgo](https://github.com/onsi/ginkgo)

```
对于支付，用例描述为
- case1
  - Given 用户存在
  - When 下单
  - Then 成功生成订单
- case2
  - Given 用户已下单
  - When 收到channel支付回调通知
  - Then 成功变更订单状态
  - Then 成功发起发货通知
```