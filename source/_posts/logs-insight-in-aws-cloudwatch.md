---
title: 在 AWS CloudWatch Logs Insights 中进行日志数据的过滤、解析
tags: 
- aws
- 编程
date: 2023-03-15 08:00:00
---

CloudWatch Logs Insights 是一项 AWS 服务，用于分析和查询存储在 CloudWatch 日志中的数据。通过使用 CloudWatch Logs Insights，您可以执行强大的查询操作来过滤、解析和分组日志数据，从而获得有关日志事件的有价值的信息。

下面是在 CloudWatch Logs Insights 中使用的2个关键操作：过滤（Filter）、解析（Parse）。

1. 过滤（Filter）：通过使用过滤器，您可以根据特定的条件筛选出感兴趣的日志事件。过滤器可以根据文本内容、时间戳、字段值等进行设置。通过过滤操作，您可以限制查询的范围，仅关注满足特定条件的日志事件。
2. 解析（Parse）：在 CloudWatch Logs Insights 中，解析操作用于从日志事件中提取特定的字段或信息。解析允许您使用模式匹配来识别和提取感兴趣的数据。您可以根据特定的模式定义解析规则，然后将匹配的数据提取到命名的字段中，以便进一步分析和使用。

这些操作可以结合使用，以构建复杂和有针对性的查询，以满足您的具体需求。

假设您有一个应用程序将一个经过字符串化的 JSON 记录到 CloudWatch 日志中，并且您有一个要对这些数据进行某种类型的分析的需求。以下是 JSON 示例：

```json
{ "request_id": "abc123", "path": "/servicea/hello", "process_time": 14.433 }
```

以下是记录这个 JSON 时的日志样式：

```plain/text
2020-06-25 INFO response from server: { "request_id": "abc123", "path": "/servicea/hello", "process_time": 14.433 }
```

假设您想要按 process_time 从大到小排序查询结构以便对请求做优化。让我们看看如何在 CloudWatch Logs Insights 中构建一个查询，以获得这个输出：

```
|-------------------|--------------|
|  path             | process_time |
|-------------------|--------------|
|  /servicea/hello  | 287.3332     |
|  /servicea/hello  | 125.98323    |
|-------------------|--------------|
```

首先，由于每个 CloudWatch 日志事件本身就是一个 JSON 对象，我们使用以下方式仅提取出日志消息：

```
fields @message
```

这样我们就可以得到所有的日志。接下来，让我们过滤掉不需要的日志语句：

```
fields @message |
filter @message like 'response from server'
```

这样我们就得到了仅包含记录请求响应的日志。

接下来，我们需要提取 process_time，以便稍后对其进行排序。使用 `parse` 命令来提取客户端 ID：

```
fields @message 
| filter @message like 'response from server'
| parse @message '"process_time":*} 'as process_time
| fields @timestamp, @message, abs(process_time) as t
| sort t desc
| limit 200
```

上述 `parse` 语句的作用是将我们记录在日志中的 process_time 字段给提取出来，当它在模式中找到类似 * 的通配符时，它将该值提取到以 "as" 后面命名的字段中。但此时是字符串类型，无法对其直接进行排序才做，还需将其换为换数值类型。

现在，我们就可以获取按照 process_time 从大到小的排序结果。
