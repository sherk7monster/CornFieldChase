---
title:  "准确率和召回率释义"
category: "model"
---

2021-03-31 记

<script async src="//busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js">
</script>
<span id="busuanzi_container_page_pv">
  阅读量&nbsp;<span id="busuanzi_value_page_pv"></span>&nbsp;次，
</span>本文约 {{ content | strip_html | strip_newlines | split: "" | size }} 字

目录
* 目录
{:toc}

## 场景

**假设:** 由 0 和 1 组成的一组样本，总个数为 8 ，0 表示正常，1 表示异常。现在针对异常样本进行预测识别。
{: .notice--info}

## 预测概率与实际

| 序号 | 预测为异常（值为1）的概率 | 实际（值为0或1） |
|:--------|:-------:|--------:|
| 1   | 0.5（<0.6，值为0）   | 0   |
| 2   | 0.1（<0.6，值为0）   | 1   |
|----
| 3   | 0.9（>0.6，值为1）   | 1   |
| 4   | 0.3（<0.6，值为0）   | 0   |
|----
| 5   | 0.8（>0.6，值为1）   | 0   |
| 6   | 0.2（<0.6，值为0）   | 1   |
|----
| 7   | 0.3（<0.6，值为0）   | 0   |
| 8   | 0.7（>0.6，值为1）   | 1   |
{: rules="groups"}

## 计算

假设预测概率 >= 0.6 的即可认定为异常（值为 1 ），则得到预测与实际的关系表。假定第一个数代表实际值，第二个数代表预测值，则：

| 预测值 | 实际值 | 个数 |
|:--------|:-------:|--------:|
| 0   | 0   | 3   |
| 0   | 1   | 2   |
|----
| 1   | 0   | 1   |
| 1   | 1   | 2   |

对应到下表即是：

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/model/forcast-actual-compare.jpg){: .align-center}

### 准确率

对角线位置为准确率，即预测值和实际值相等的情况，则此时准确率为：

准确率 = ( 3 + 2 ) / 8 = 0.625

### 召回率

召回率是预测出的真实异常个数（就是对角线预测和实际都为1的场景）比上实际异常个数（实际第二列的个数和）：

召回率 = 2 / ( 2 + 2 ) = 0.5

### 准确率和召回率的关系（P-R曲线）

当准确率上升的时候，召回率就不变或降低

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/model/precision-recall.jpg){: .align-center}

P：Precision , R：Recall
{: .notice--info}