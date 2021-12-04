### Prometheus 简介

[toc]

#### 教程目标

- 对 promethes 的整体架构有一个感性的认识
- 理解 prometheus 基本的数据采集过程
- 理解 prometheus 基本的数据存储方式
- 理解 PromQL 的内在逻辑
-  最终，能够尽量无痛的使用 prometheus

#### 环境准备



#### 整体架构

![Prometheus architecture](./assets/architecture.png)



1. pull 模型，prometheus 主动通过 http 请求收集指标
2. prometheus 把定期收集到的指标存成时间序列
3. prometheus 通过一个自己的 http 服务接收 promQL 对外提供数据查询服务
4. prometheus 通过配置的预聚合规则，通过已有时间序列生成新的时间序列
5. prometheus 通过配置的报警规则推送报警信息给 alertmanager
6. prometheus 通过自己的 http 服务提供自身维护使用的 api

#### exporter(node_exporter 为例)

- 作用： 暴露监控指标给 prometheus

- 获取指标： ` curl http://127.0.0.1:9100/metrics > node_exporter.metrics `

- 获取指标： ` curl http://127.0.0.1:9090/metrics > prometheus.metrics `

- 指标类型：

  - counter: 递增计数器

    ```
    # HELP promhttp_metric_handler_requests_total Total number of scrapes by HTTP status code.
    # TYPE promhttp_metric_handler_requests_total counter
    promhttp_metric_handler_requests_total{code="200"} 71842
    promhttp_metric_handler_requests_total{code="500"} 0
    promhttp_metric_handler_requests_total{code="503"} 0
    ```

  - gauge: 可增可减的值

    ```
    # HELP go_goroutines Number of goroutines that currently exist.
    # TYPE go_goroutines gauge
    go_goroutines 10
    ```

  - histogram: 直方图

    ```
    # HELP prometheus_http_request_duration_seconds Histogram of latencies for HTTP requests.
    # TYPE prometheus_http_request_duration_seconds histogram
    prometheus_http_request_duration_seconds_bucket{handler="/",le="0.1"} 1
    prometheus_http_request_duration_seconds_bucket{handler="/",le="0.2"} 1
    prometheus_http_request_duration_seconds_bucket{handler="/",le="0.4"} 1
    prometheus_http_request_duration_seconds_bucket{handler="/",le="1"} 1
    prometheus_http_request_duration_seconds_bucket{handler="/",le="3"} 1
    prometheus_http_request_duration_seconds_bucket{handler="/",le="8"} 1
    prometheus_http_request_duration_seconds_bucket{handler="/",le="20"} 1
    prometheus_http_request_duration_seconds_bucket{handler="/",le="60"} 1
    prometheus_http_request_duration_seconds_bucket{handler="/",le="120"} 1
    prometheus_http_request_duration_seconds_bucket{handler="/",le="+Inf"} 1
    prometheus_http_request_duration_seconds_sum{handler="/"} 4.0507e-05
    prometheus_http_request_duration_seconds_count{handler="/"} 1
    ```

  - summary: 分位数

    ```
    # HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
    # TYPE go_gc_duration_seconds summary
    go_gc_duration_seconds{quantile="0"} 2.71e-05
    go_gc_duration_seconds{quantile="0.25"} 5.4368e-05
    go_gc_duration_seconds{quantile="0.5"} 0.000151666
    go_gc_duration_seconds{quantile="0.75"} 0.000257266
    go_gc_duration_seconds{quantile="1"} 0.002850318
    go_gc_duration_seconds_sum 42.687752096
    go_gc_duration_seconds_count 159934
    ```

#### PromQL 

时间序列示意图， 每个时间序列都是由指标名字和标签值确定的（事实上指标名字也是一个特殊标签）



![time-series.png](/home/zhngcho/Documents/comeaross-promethues/assets/time-series-arrow.svg)

​			

##### 基础查询

1. 瞬时查询

   - 示意图

     ![promql-vector](/home/zhngcho/Documents/comeaross-promethues/assets/promql-instant-vector.svg)

   - 语法 `promhttp_metric_handler_requests_total{code="200"} @ 624669000000 offset 5m`

   - 结果为一个向量

     ```
     promhttp_metric_handler_requests_total{code="200", instance="localhost:9090", job="prometheus"}
     77552
     promhttp_metric_handler_requests_total{code="200", instance="localhost:9100", job="node"}
     ```

2. 区间查询

   - 示意图

     ![promql-range-vector](/home/zhngcho/Documents/comeaross-promethues/assets/promql-range-vector.svg)

   - 语法 `promhttp_metric_handler_requests_total{code="200"} [2m] @ 624669000000 offset 5m`

   - 结果为区间向量

     ```
     promhttp_metric_handler_requests_total{code="200", instance="localhost:9090", job="prometheus"}
     77639 @1634823906.889
     77640 @1634823921.889
     77641 @1634823936.889
     77642 @1634823951.889
     77643 @1634823966.889
     77644 @1634823981.889
     77645 @1634823996.889
     77646 @1634824011.889
     promhttp_metric_handler_requests_total{code="200", instance="localhost:9100", job="node"}
     77647 @1634823909.821
     77648 @1634823924.821
     77649 @1634823939.821
     77650 @1634823954.821
     77651 @1634823969.821
     77652 @1634823984.821
     77653 @1634823999.821
     77654 @1634824014.821
     ```

3. 其他说明， 以下均为合法表达式

   - 无offset, `promhttp_metric_handler_requests_total{code="200"}`
   - 无 metrics name, `{__name__ = "promhttp_metric_handler_requests_total", code="200"}`
   - 无 labels, `promhttp_metric_handler_requests_total`
   - label = value, label != value, label =~ reg, label !~ reg 都支持（reg 不能匹配空字符串）



##### 简单运算

- 算术运算符: `+ - * / % ^`
  - 操作数: 
    - 标量&标量`1+2`
    - 标量&向量 `promhttp_metric_handler_requests_total{code="200"} + 1`
    - 向量&标量 `1+promhttp_metric_handler_requests_total{code="200"}`
    - 向量&向量 `promhttp_metric_handler_requests_total{code="200"} - promhttp_metric_handler_requests_total{code="200"} offset 5m`
- 比较运算符 `== != > < >= <=`
  - 操作数
    - 标量&标量`1 > bool 2`
    - 向量&标量 `promhttp_metric_handler_requests_total{code="200"} > 1`
    - 向量&标量 `promhttp_metric_handler_requests_total{code="200"} > bool 1`
    - 标量&向量 `1 > promhttp_metric_handler_requests_total{code="200"}`
    - 标量&向量 `1 > bool promhttp_metric_handler_requests_total{code="200"}`
    - 向量&向量 `promhttp_metric_handler_requests_total{code="200"} > promhttp_metric_handler_requests_total{code="200"} offset 5m`
    - 向量&向量 `promhttp_metric_handler_requests_total{code="200"} > bool promhttp_metric_handler_requests_total{code="200"} offset 5m`
  - 以上 bool 为修饰符，无 bool 则表示过滤时间序列，有 bool 则表示时间序列的计算（参考瞬时查询示意图）
- 集合运算符 `and or unless`
  - 操作数
    - 向量&向量 `promhttp_metric_handler_requests_total{code="200"} and {instance =~ ".*9100"}`
  - 整体运算方式为以左操作数为主，根据不同的运算，在左操作数中进行添加和删除元素
- 向量匹配规则
  - 一对一
    - `prometheus_http_requests_total{code="200", handler="/api/v1/query", instance="localhost:9090", job="prometheus"} / ignoring(handler) promhttp_metric_handler_requests_total{code="200", instance="localhost:9090", job="prometheus"}`
  - 一对多
    - `promhttp_metric_handler_requests_total{instance="localhost:9090", job="prometheus"}/ignoring(handler) group_right  prometheus_http_requests_total{code="200",  instance="localhost:9090", job="prometheus"} `
  - 多对一
    - `prometheus_http_requests_total{code="200",  instance="localhost:9090", job="prometheus"} / ignoring(handler) group_left promhttp_metric_handler_requests_total{instance="localhost:9090", job="prometheus"}`
  - 注意： *Many-to-one and one-to-many matching are advanced use cases that should be carefully considered. Often a proper use of `ignoring(<labels>)` provides the desired outcome.*

##### 时间序列的聚合

###### 瞬时向量的聚合

- `sum count min max avg quantile stddev stdvar topk bottomk ... `

- `sum (prometheus_http_requests_total{ instance="localhost:9090", job="prometheus"}) without (code)`
- `sum (prometheus_http_requests_total{ instance="localhost:9090", job="prometheus"}) by (code)`

###### 区间向量的聚合（\<aggregation\>_over_time())

- `avg_over_time(prometheus_http_requests_total [5m])`

##### 其他函数

`ceil(v instant-vector) irate(v range-vector) ...`

##### 一些有意思的问题

- 时间序列 foo [(0, 23), (15, 50), (30, 45), (45, 8), (60, 10), ...]，foo @ 17 的结果是什么？

  ```shell
  echo 'rule_files:' > test.yml
  echo 'evaluation_interval: 1s'>> test.yml
  echo 'tests:' >> test.yml
  echo '- interval: 15s' >> test.yml
  echo '  input_series:'>> test.yml
  echo '  - series: foo'>> test.yml
  echo '    values: 23 50 45 8 10'>> test.yml
  echo '  promql_expr_test:'>> test.yml
  echo '  - expr: foo'>> test.yml
  echo '    eval_time: 17s'>> test.yml
  echo '    exp_samples:'>> test.yml
  echo '    - labels: "foo{}"' >> test.yml
  echo '      value: 50' >> test.yml
  promtool test rules test.yml
  ```

- 再来一次，时间序列 foo [(0, 23), (15, 50), (30, 45), (45, 8), (60, 10), ...]，foo @ 17 的结果是什么？

  ```shell
  echo 'rule_files:' > test.yml
  echo 'evaluation_interval: 10s'>> test.yml
  echo 'tests:' >> test.yml
  echo '- interval: 15s' >> test.yml
  echo '  input_series:'>> test.yml
  echo '  - series: foo'>> test.yml
  echo '    values: 23 50 45 8 10'>> test.yml
  echo '  promql_expr_test:'>> test.yml
  echo '  - expr: foo'>> test.yml
  echo '    eval_time: 17s'>> test.yml
  echo '    exp_samples:'>> test.yml
  echo '    - labels: "foo{}"' >> test.yml
  echo '      value: 50' >> test.yml
  promtool test rules test.yml
  ```

  
