### 概念
- 分布式服务时代，服务之间的请求域调用不再是简单的直连方式，注册中心的出现，让服务治理更加便利，也对服务之间的链路追踪提出了更高的要求
- 将一次分布式请求还原成调用链路，将一次分布式请求的调用情况集中展示
- 比较主流的链路追踪产品都是启发于google发表的Dapper，Dapper阐述了分布式系统，特别是微服务架构中链路追踪的概念、数据表示、埋点、传递、收集、存储与展示等技术细节
### 作用
- 故障快速定位
- 各个调用环节的性能分析
- 数据分析
- 生成服务调用拓扑图
## 追踪理论Dapper
- Trace的含义比较直观，就是链路，指一个请求经过后端所有服务的路径，每一条链路都用一个全局唯一的traceid来标识
- span之间存在着父子关系，上游的span是下游的父span，每个span由spanid和parentid来标识，spanid在一条链路中唯一
- 通用架构
    - 采样、收集、存储与展示
      ![image](https://user-images.githubusercontent.com/31843331/133041048-67af07d1-6f9a-44a1-8aa6-e24d3b71f946.png)
    - 主流的框架对比图
      - dapper论文中提到的tracing system  Magpie X-Trace
      ![image](https://user-images.githubusercontent.com/31843331/133042094-c3aea08e-2796-4a7c-a54a-129bd5d3fe98.png)

## Jeager简介
- Jaeger
  ![image](https://user-images.githubusercontent.com/31843331/133052007-11ddb378-e6ec-4855-96ad-c9c3f99b2750.png)
- Jaeger-Client
    - 实现了OpenTracing API接口
    - 当应用建立Span并发出请求到下游的服务时，它会附带跟踪信息(Trace ID, Span ID, baggage)
    - 默认比例是0.1%的取样
    - 跟踪功能是默认开启

## 特点
- Jaeger选择Go作为主要语言，Go是作为一种系统语言编写的
- Jaeger虽然缺乏成熟度，但它具有速度快和灵活性高的特点，还有更高的性能，并且更易于扩展

## 服务器搭建
- docker搭建服务器
```shell
docker run   --rm   -p 6831:6831/udp   -p 6832:6832/udp   -p 16686:16686   jaegertracing/all-in-one:1.7   --log-level=debug
```

## 初始化
- 配置
```go
	cfg := &config.Configuration{
		ServiceName: service,
		Sampler: &config.SamplerConfig{
			Type:  "const",
			Param: 1,
		},
		Reporter: &config.ReporterConfig{
			LogSpans: true,
			LocalAgentHostPort: "172.1.1.156:6831",
		},
	}
```
- tracer创建
```go
	tracer, closer, err := cfg.NewTracer(config.Logger(jaeger.StdLogger))
```

### 使用1
- 创建span
```go
	span := tracer.StartSpan("say-hello-uber")

```
- 标签
```go
	span.SetTag("hello-to-uber", helloTo)
```
- tag
```go
	span.SetTag("hello-to-uber", helloTo)
```

- log 
```go
	span.LogFields(
		log.String("event", "string-format"),
		log.String("value", helloStr),
	)

span.LogKV("event", "println")

```
### 使用2  调用函数间追踪 通过上下文
- 调用函数
```go
	ctx := opentracing.ContextWithSpan(context.Background(), span)
```
- 被调用函数
```go
	span, _ := opentracing.StartSpanFromContext(ctx, "formatString")
```
### 使用3 服务之间的传递(http)  Inject->Extract
- 请求端
```go
	ext.SpanKindRPCClient.Set(span)
	ext.HTTPUrl.Set(span, url)
	ext.HTTPMethod.Set(span, "GET")
	span.Tracer().Inject(
		span.Context(),
		opentracing.HTTPHeaders,
		opentracing.HTTPHeadersCarrier(req.Header),
	)
```
- 被请求端
```go
		spanCtx, _ := tracer.Extract(opentracing.HTTPHeaders, opentracing.HTTPHeadersCarrier(r.Header))
```
### Baggage item 元数据,实现服务间值传递
- 请求端
```go
	span.SetBaggageItem("greeting", greeting)
```
- 被请求端
```go
	greeting := span.BaggageItem("greeting")
```
## 参考文档（jaeger）
- [环境搭建](http://www.bubuko.com/infodetail-3421585.html)
- [官方github](https://github.com/jaegertracing/jaeger-client-go)
- [官网](https://opentracing.io/guides/golang/)
- [文档](https://blog.csdn.net/weixin_34393428/article/details/92408030)  

## 参考文档（zipkin）
- [Dapper论文](https://research.google/pubs/pub36356/)
- [官网](https://zipkin.io/pages/tracers_instrumentation.html)
- [github-zipkin](https://github.com/openzipkin/zipkin-go)
- [安装方法](https://github.com/openzipkin/zipkin)
- [Open-Tracing](https://wu-sheng.gitbooks.io/opentracing-io/content/pages/spec.html)
- [Open-Tracing-github](https://github.com/opentracing/opentracing-go)

