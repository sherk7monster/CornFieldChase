---
title:  "Leetcode-两两交换链表中的节点解题分析"
category: "algorithm"
---

###### 2020-09-09 记

<script async src="//busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js">
</script>
<span id="busuanzi_container_page_pv">
  阅读量&nbsp;<span id="busuanzi_value_page_pv"></span>&nbsp;次
</span>

## 题目描述

```
原链表：  1->2->3->4->5->6
返回：    2->1->4->3->6->5
```

## 解题分析

**分析:** 2个成对操作，(2->1)，(4->3)，(6->5)，每次操作时将上一个节点的next指针指向成对节点的第一个节点，如：1->4，3->6依次迭代
{: .notice--info}

## 画图分析

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/swapnodes.jpeg){: .align-center}

## 迭代的代码实现

```java
public class SwapTwoNodesInPairs {
    public ListNode swapPairs(ListNode head) {
        if(head==null||head.next==null){
            return head;
        }
        ListNode prev=head;
        ListNode cur=head;
        head=head.next;
        while (cur!=null&&cur.next!=null){//结束条件
            ListNode temp1=cur.next.next;
            //这里将前置节点的next指向当前节点的next节点，并且交换当前节点和当前节点的next节点
            ListNode temp2=cur.next;
            prev.next=temp2;
            cur.next=temp1;
            temp2.next=cur;
            //将前置节点指针指向当前节点（当前节点交换后位置会在后面）
            prev=cur;
            cur=temp1;
        }
        return head;
    }

    public class ListNode {
        int val;
        ListNode next;
        ListNode(int x) { val = x; }
    }
}
```