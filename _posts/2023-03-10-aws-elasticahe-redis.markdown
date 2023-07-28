---
layout: post
title:  "Serverless: (Go) Cấu hình AWS ElastiCache Redis và tích hợp với AWS Lambda"
date:   2023-03-10 11:00:00 +0700
tags: serverless golang
categories: serverless
---
\
Redis là giải pháp cho việc lưu trữ data NoSQL, ngoài ra còn dùng để cache data rất hữu ích, và còn nhiều công dụng tuyệt vời khác nữa. Trong bài post này, tôi sẽ hướng dẫn bạn cấu hình AWS ElastiCache Redis và tích hợp với AWS Lambda bằng Go.

## **Cấu hình Redis (cluster mode disabled) bằng AWS Console**
---
\
Các bước làm như sau:
1. Mở trang ElastiCache Dashboard 
2. Chọn **Get started** 
3. Chọn **Create cluster**
4. Chọn **Create Redis cluster**
5. Trong mục **Cluster settings**, làm theo các bước sau:

    a. Chọn **Configure and create a new cluster**\
    b. **Cluster mode**: chọn **Disabled**\
        &nbsp;&nbsp;&nbsp;&nbsp;
        _Note: Disabled có nghĩa là Redis cluster sẽ có 1 shard(node group). Thực tế bên trong 1 shard vẫn sẽ có primary node và tối đa có 5 replica node [(AWS docs)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/CacheNodes.NodeGroups.html)_\
    c. **Cluster info**: Nhập **Name**: "els-redis", **Description**: "redis cache for test"\
    d. **Location**: chọn **AWS Cloud**, chọn **Enable Multi-AZ**\
        &nbsp;&nbsp;&nbsp;&nbsp;
        _Note: **On premises** tức là thông qua **AWS Outposts** để build và run các ứng dụng trên local compute on premises [(AWS docs)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/ElastiCache-Outposts.html). **Multi-AZ** sẽ giúp hệ thống tránh bị downtime, với cơ chế fail over, khi primary node gặp vấn đề sẽ tự động chuyển 1 replica node thành primary node [(AWS docs)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/AutoFailover.html)_\
    e. **Cluster settings**:\
        &nbsp;&nbsp;&nbsp;&nbsp;
        **Port**: 6379\
        &nbsp;&nbsp;&nbsp;&nbsp;
        **Node type**: cache.t3.micro\
            &nbsp;&nbsp;&nbsp;&nbsp;
            _Note: Khi lựa chọn node type cho **Redis (cluster mode disabled) cluster** cần lưu ý:_
            * _Nếu bạn ước tính tổng dung lượng bạn cần lưu trữ là A GB, thì dung lượng bộ nhớ cần có phải là 2A GB. Lý do là Redis cần thêm vùng nhớ cho việc vận hành **BGSAVE** [(AWS docs)](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/nodes-select-size.html)._\
        &nbsp;&nbsp;&nbsp;&nbsp;
        **Number of replicas**: 1\
    f. **Subnet group settings**: chọn **Create a new subnet group**, nhập Name và Description, chọn **VPC ID** và chọn các subnet.\
    g. **Availability Zone placements** chọn **No preference**\
    h. Chọn **Next**\

6. Trong mục **Advanced settings**, làm theo các bước sau:

    a. Tạo **Security groups** với inbound là port 6379 để cho phép connect tới Redis qua port 6379\
    b. **Backup** bỏ chọn **Enable automatic backups**\
    c. **Maintenance** chọn **No preference**\
    d. Chọn **Next**

7. Review lại 1 lượt và chọn **Create**

## **Tích hợp AWS Elasticache Redis với Go**
---
\
Trong phần tiếp theo, chúng ta sẽ tích hợp Elasticache Redis vào Go để test thử trên AWS Lambda nhé.

* Đầu tiên, tôi sẽ tạo package redis để khởi tạo kết nối tới Redis, cũng như implement các method cần thiết cho Redis như: Set, Get, Del,... Package redis này sẽ được dùng cho các external service, nên tôi sẽ khai báo interface trong chính package này (producer side).

{% highlight go %}
// RedisCache is the interface implemented for Redis.
// This interface in producer side because it will be expose for external service.
type RedisCache interface {
	Exists(ctx context.Context, key string) (bool, error)
	Set(ctx context.Context, key string, value interface{}, expiration time.Duration) error
	Get(ctx context.Context, key string, valueType interface{}) error
	Del(ctx context.Context, key string) error
	Expire(ctx context.Context, key string, expiration time.Duration) error
	SetNX(ctx context.Context, key string, value interface{}, expiration time.Duration) (bool, error)
}

type redisCache struct {
	client *go_redis.Client
}

// NewRedisCache creates an instance.
func NewRedisCache(addrs string) RedisCache {
	client, err := initRedis(addrs)
	if err != nil {
		panic(fmt.Errorf("failed to init redis with err: %w", err))
	}
	return &redisCache{
		client: client,
	}
}

func initRedis(addr string) (*go_redis.Client, error) {
	client := go_redis.NewClient(&go_redis.Options{
		Addr: addr,
		DB:   0,
	})
	_, err := client.Ping(context.Background()).Result()
	return client, err
}

// Exists to check key has been already existed.
//
// Return: true: is existed, false: not exist.
func (h *redisCache) Exists(ctx context.Context, key string) (isExisted bool, err error) {
	result, err := h.client.Exists(ctx, key).Result()
	if err != nil {
		return true, err
	}

	// result = 0 is means key does not exist
	if result == 0 {
		return false, nil
	}
	return true, nil
}

// Set to save key and value.
func (r *redisCache) Set(ctx context.Context, key string, value interface{}, expiration time.Duration) (err error) {
	data, err := json.Marshal(value)
	if err != nil {
		return err
	}
	_, err = r.client.Set(ctx, key, string(data), expiration).Result()
	if err != nil {
		return err
	}
	return nil
}

// Get to get value by key. Need to pass pointer of valueType.
func (r *redisCache) Get(ctx context.Context, key string, valueType interface{}) (err error) {
	data, err := r.client.Get(ctx, key).Result()
	if err != nil {
		return err
	}
	err = json.Unmarshal([]byte(data), &valueType)
	if err != nil {
		return err
	}

	return nil
}

// Del to delete key.
func (h *redisCache) Del(ctx context.Context, key string) (err error) {
	_, err = h.client.Del(ctx, key).Result()
	if err != nil {
		return err
	}
	return nil
}

// Expire to set more expiration for the key.
func (h *redisCache) Expire(ctx context.Context, key string, expiration time.Duration) (err error) {
	_, err = h.client.Expire(ctx, key, expiration).Result()
	if err != nil {
		return err
	}
	return nil
}

// SetNX to set key and value.
// Return the result:
//
//	true: if key does not exist, SetNX is successfully.
//	false: if key has been already existed, SetNX failed
func (h *redisCache) SetNX(ctx context.Context, key string, value interface{}, expiration time.Duration) (result bool, err error) {
	data, err := json.Marshal(value)
	if err != nil {
		return false, err
	}

	return h.client.SetNX(ctx, key, string(data), expiration).Result()
}
{% endhighlight %}

* Sau khi đã có package redis, tôi sẽ tạo 1 package helper cho redis. Do các method trong package redis tôi dùng kiểu dữ liệu interface{}, nên package helper sẽ giúp quản lý việc cast giữ liệu sang các model struct khác nhau 1 cách thuận tiện.

{% highlight go %}

type RedisHelper struct {
	RedisCache redis.RedisCache
}

// NewRedisCache creates an instance
func NewRedisHelper(addrs string) *RedisHelper {
	return &RedisHelper{
		RedisCache: redis.NewRedisCache(addrs),
	}
}

func (r *RedisHelper) SetDataExample(ctx context.Context, key string, merchantKey model.DataExample,
	expiration time.Duration) (err error) {
	err = r.RedisCache.Set(ctx, key, merchantKey, expiration)
	if err != nil {
		return fmt.Errorf("failed to set data example to Redis with err: %w", err)
	}
	return nil
}

func (r *RedisHelper) GetDataExample(ctx context.Context, key string) (dataEx *model.DataExample, err error) {
	dataEx = &model.DataExample{}
	err = r.RedisCache.Get(ctx, key, dataEx)
	if err != nil {
		return dataEx, fmt.Errorf("failed to get data example from Redis with err: %w", err)
	}
	return dataEx, nil
}

func (r *RedisHelper) CheckDataExampleExisted(ctx context.Context, key string) (isExisted bool, err error) {
	isExisted, err = r.RedisCache.Exists(ctx, key)
	if err != nil {
		return false, fmt.Errorf("failed to check key exist in Redis with err: %w", err)
	}
	return isExisted, nil
}

func (r *RedisHelper) SetNXDataExample(ctx context.Context, key string, merchantKey model.DataExample,
	expiration time.Duration) (result bool, err error) {
	result, err = r.RedisCache.SetNX(ctx, key, merchantKey, expiration)
	if err != nil {
		return false, fmt.Errorf("failed to set nx data example in Redis with err: %w", err)
	}
	return result, nil
}

func (r *RedisHelper) DeleteDataExample(ctx context.Context, key string) (err error) {
	err = r.RedisCache.Del(ctx, key)
	if err != nil {
		return fmt.Errorf("failed to delete data example in Redis with err: %w", err)
	}
	return nil
}
{% endhighlight %}

* Tiếp đến là package main, tại đây tôi sẽ tạo 1 service để khởi tạo các dependency cho package main, cũng như khai báo các method mà package muốn sử dụng vào interface.

{% highlight go %}
type service struct {
	redisCache DataExampleRedisCache
}

// NewService create new instance.
func NewService(addrs string) service {
	redisHelper := redishelper.NewRedisHelper(addrs)
	return service{
		redisCache: redisHelper,
	}
}

// This interface in consumer side because package main just want to use 4 method.
type DataExampleRedisCache interface {
	SetDataExample(ctx context.Context, key string, dataEx model.DataExample, expiration time.Duration) (err error)
	GetDataExample(ctx context.Context, key string) (dataEx *model.DataExample, err error)
	CheckDataExampleExisted(ctx context.Context, key string) (isExisted bool, err error)
	SetNXDataExample(ctx context.Context, key string, dataEx model.DataExample, expiration time.Duration) (result bool, err error)
	DeleteDataExample(ctx context.Context, key string) (err error)
}
{% endhighlight %}

* Và cuối cùng, tại func main tôi sẽ khởi tạo instance service mới và test thử.

{% highlight go %}
s := NewService(addrRedis)
cacheKeyFirst := "cache_key_first"
dataExample := model.DataExample{
    ID:       1,
    Name:     "first",
    IsActive: true,
}
testSetDataExample(ctx, s, cacheKeyFirst, dataExample)
testGetDataExample(ctx, s, cacheKeyFirst)
testCheckExisted(ctx, s, cacheKeyFirst)
{% endhighlight %}

[**Full source code**](https://github.com/quantr247/go-redis-example)

## **Tổng kết**
---
\
Với việc triển triển khai Redis trong hệ thống, chúng ta có thể đạt được lợi ích:
* NoSQL database để lưu trữ dữ liệu
* Caching data, giúp giảm thời gian duration của hệ thống.
* Giải quyết bài toán race condition với SetNX

Tuy nhiên, tùy thuộc vào use case, cũng như yêu cầu cụ thể mà bạn cần điều chỉnh lại cho phù hợp với hệ thống của mình.

Chúc bạn làm việc hiệu quả :)

## **Tài liệu tham khảo**
---
\
[Create AWS ElastiCache Redis cluster](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/GettingStarted.CreateCluster.html#Clusters.Create.CON.Redis-gs)

[AWS SDK for Go API integrate Elasticache](https://docs.aws.amazon.com/sdk-for-go/api/service/elasticache/)

[Redis library for Go](https://github.com/redis/go-redis)