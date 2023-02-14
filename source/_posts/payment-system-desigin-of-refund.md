---
title: 聚合支付系统设计之：退款处理
description: 退款是一个相当重要功能，但由于从支付到申请退款的时间跨度长、流程中不确性多、过程异步化程度高，应此在设计退款功能时应妥善考虑各种意外情况以保证退款顺利完成。
keyword: 聚合支付,支付,退款,系统设计
tags: 
- 系统架构设计
- 支付系统
date: 2023-01-06 08:00:00
---

## 概览

顾客在商家的系统中完成付款操作后，还可以在一定时间内申请退款。因此聚合支付系统需要面向商家端提供退款能力以便商户系统完成相应操作。

## 退款状态机

以下为退款单的状态转换图：

![取合支付退款状态图](https://oss-digitcert.oss-cn-shenzhen.aliyuncs.com/uPic/取合支付退款状态图.png)

Processing： 商户系统向聚合支付系统申请退款成功

Failed: 聚合支付系统向收单机构申请退款，收单机构返回失败，常见的原因：商家账户或用户账户异常等

Closed: 因订单超过退款期限，或商家商户余额不足等明确的业务原因失败，此种原因的失败可发起重试

Success: 退款成功

## 退款处理流程

![取合支付退款时序图](https://oss-digitcert.oss-cn-shenzhen.aliyuncs.com/uPic/取合支付退款时序图.png?3)

### 申请退款

聚合支付系统仅处理支付业务，因此退款需要顾客在商户系统中提出申请，或由商户系统具备权限的运营人员发起，并在商户系统创建退款单。然后向聚合支付系统提出申请，聚合支付系统创建退款单后返回单号给商户系统。

对于聚合支付系统而言，退款是一个纯异步的操作。退款单创建成功后进入系统处理队列，由处理程序向收单机构发起退款。

### 查询退款

聚合支付系统提供了一组退款接口，其中包括：申请退款接口以及查询退款接口。

申请退款接口所返回的结果仅代表该次退款请求是否受理成功（含聚合支付系统的退款单号）。由于本身退款的结果部分依赖于用户支付卡银行的处理，所以最终退款结果是否成功还需要通过查询退款接口来确认。或接收来自聚合支付系统的退款结果通知（退款单有状态变化时，聚合支付系统将向商家系统发送退款通知）。

### 退款时效

退款的操作具有相当的复杂性、操作都是异步化的特点，同时不同的收单机构对退款的处理时间也不尽相同：

	- 支付宝、微信支付余额类的支付通常都能在 30 分钟内处理完毕
	- 使用绑定银行卡支付，退款处理需要 0-5 个工作日
	- 如果需要重试，则会消耗更多时间

鉴于退款时间的不确定性，当顾客在商户系统中申请退款成功能时，建议系统向顾客告知退款受理状态，对于款项的到帐时间应预留一定的处理时间，比如向顾客提示退款将会3～7个工作日内按支付方式原路返回并提醒顾客关注付款渠道的通知是比较妥当的作法。

### 退款重试

在上面的状态图中所描述的关闭状态（Closed），商户系统可以进行重试退款。但切记，重试时请**使用上次相同的退款单号**。

### 多次退款

这是因为收单机构都允许对一个交易进行部分退款或多次退款，只要退款总额不超过付款金额就行。在聚合支付系统及收单机构系统中，都将不同的退款单号认定为不同的退款，即使两笔退款发生在同一笔交易下。

这也就解释了在进行重试退款时为什么不能变更退款单号。

在部分退款的场景，多次以不同单号重试就可能会造重复退款的情况。即使上次退款没成功能，变更了退款单号并虽不会导致重复退款，但这在语义上也有混淆：到底是申请了2次退款，还是只申请了1次因失败又重试了1次？无法解释。

### 正确做法

重试时不改变退款单号，但增加退款序号，用于标记处理此笔退款的多次处理。

### 余额不足

收单机构给聚合支付系统返回余额不足错误时，聚合支付系统将会给商户系统返回 ** Close 状态**，且原因被标记为 "NOT_ENOUGH"，此时商户系统可在稍后重试。

### 什么情况下会出现 “NOT ENOUGH”

当商户在收单机构的账户资金池余额不足时。

拿微信支付来说，商户与微信支付签约后会有一个资金结算周期，实体行业为 T+1、互联网行业为 T+7 等。如果为 T+1 结算，在每日的凌晨，微信支付会将商户收到的资金结算到商户的结算账户中。

微信支付的资结算是自动的，无需人工手动申请提现。那么一个订单在付款3天后来申请退款，恰好最近3天商户系统没有交易产生，此时商户在微信支付的待结算的资金池余额为0，申请退款就会无款可退，导致失败。

## 完整退款逻辑与解决方案

1. 退款要支持重试，并且重试时不能改变退款单号;
2. 一笔交易支持多次退款，商户系统、聚合支付系统做好校验与检查;
3. 退款的重试动作，应该留在聚合支付系统;
4. 退款是异步操作，结果应以异步通知为主;
5. 前端对顾客友好提示，告知顾客关注支付渠道的退款通知;
6. 商户应采取一定的措施保证其在收单机构的账户中有足够的待结算资金，以便覆盖可能的退款情况。