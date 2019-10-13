---
layout: post
title: "Prometheus 指标类型"
description: "prometheus metric types"
category: 
tags: []
---
{% include JB/setup %}

本文主要记录对Prometheus 指标类型的理解。

#### Counter
- - -
Counter 类型可以认为是一个单调递增的计数器，只有在初始的时候为0，之后不断递增。例如在业务服务里记录处理的请求量，任务系统里任务完成的数量等。

在程序中metric_path 输出的信息如下：

```plaintext
# HELP rpc_request rpc request count
# TYPE rpc_request counter
rpc_request{type="bar"} 2044.0
rpc_request{type="foo"} 2095.0
```

prometheus client_go 提供的NewCounterVec 和NewCounter 区别主要在于前者可灵活设置Label。

#### Gauge
- - -
Gauge 类型用于表示一个会上下浮动的数值，比如温度，系统运行的进程数目，服务连接数等。

在程序中metric_path 输出的信息如下：

```plaintext
# HELP rpc_conn rpc conn count
# TYPE rpc_conn gauge
rpc_conn{type="bar"} 24.0
rpc_conn{type="foo"} 19.0
```

#### Histogram
- - -
Histogram 类型对观测到的值按配置好的桶进行计数。

```plaintext
# HELP rpc_durations_histogram_seconds RPC latency distributions.
# TYPE rpc_durations_histogram_seconds histogram
rpc_durations_histogram_seconds_bucket{le="-0.00099"} 0.0
rpc_durations_histogram_seconds_bucket{le="-0.00089"} 0.0
rpc_durations_histogram_seconds_bucket{le="-0.0007899999999999999"} 0.0
rpc_durations_histogram_seconds_bucket{le="-0.0006899999999999999"} 0.0
rpc_durations_histogram_seconds_bucket{le="-0.0005899999999999998"} 1.0
rpc_durations_histogram_seconds_bucket{le="-0.0004899999999999998"} 1.0
rpc_durations_histogram_seconds_bucket{le="-0.0003899999999999998"} 8.0
rpc_durations_histogram_seconds_bucket{le="-0.0002899999999999998"} 17.0
rpc_durations_histogram_seconds_bucket{le="-0.0001899999999999998"} 37.0
rpc_durations_histogram_seconds_bucket{le="-8.999999999999979e-05"} 73.0
rpc_durations_histogram_seconds_bucket{le="1.0000000000000216e-05"} 123.0
rpc_durations_histogram_seconds_bucket{le="0.00011000000000000022"} 169.0
rpc_durations_histogram_seconds_bucket{le="0.00021000000000000023"} 207.0
rpc_durations_histogram_seconds_bucket{le="0.0003100000000000002"} 225.0
rpc_durations_histogram_seconds_bucket{le="0.0004100000000000002"} 236.0
rpc_durations_histogram_seconds_bucket{le="0.0005100000000000003"} 237.0
rpc_durations_histogram_seconds_bucket{le="0.0006100000000000003"} 238.0
rpc_durations_histogram_seconds_bucket{le="0.0007100000000000003"} 239.0
rpc_durations_histogram_seconds_bucket{le="0.0008100000000000004"} 239.0
rpc_durations_histogram_seconds_bucket{le="0.0009100000000000004"} 239.0
rpc_durations_histogram_seconds_bucket{le="+Inf"} 239.0
rpc_durations_histogram_seconds_sum 0.0002787468304025881
rpc_durations_histogram_seconds_count 239.0
```

#### Summary
- - -
Summary 类型对观测到的值按配置好的百分位数进行统计。

```plaintext
# HELP rpc_durations_seconds RPC latency distributions.
# TYPE rpc_durations_seconds summary
rpc_durations_seconds{service="exponential",quantile="0.5"} 6.614584593229785e-07
rpc_durations_seconds{service="exponential",quantile="0.9"} 2.0356270770924523e-06
rpc_durations_seconds{service="exponential",quantile="0.99"} 4.9881631088716e-06
rpc_durations_seconds_sum{service="exponential"} 0.0003348821685824868
rpc_durations_seconds_count{service="exponential"} 358.0
rpc_durations_seconds{service="normal",quantile="0.5"} 1.6621323314813106e-06
rpc_durations_seconds{service="normal",quantile="0.9"} 0.00025830474325216625
rpc_durations_seconds{service="normal",quantile="0.99"} 0.0004120713077157963
rpc_durations_seconds_sum{service="normal"} 0.0002787468304025881
rpc_durations_seconds_count{service="normal"} 239.0
rpc_durations_seconds{service="uniform",quantile="0.5"} 8.47106307771682e-05
rpc_durations_seconds{service="uniform",quantile="0.9"} 0.00017303499949665728
rpc_durations_seconds{service="uniform",quantile="0.99"} 0.00019709311572664958
rpc_durations_seconds_sum{service="uniform"} 0.01676565180060285
rpc_durations_seconds_count{service="uniform"} 179.0
```
