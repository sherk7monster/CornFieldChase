---
title:  "Leetcode-有效的字母异位词解题分析"
---

## 题目描述

```
示例 1:
    输入: s = "a#fg", t = "#afg"
    输出: true
```

## 解题分析

**分析:** 这个跟猜数字题目类似，但是得用map存unicode字符
{: .notice--info}

## 画图分析

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/validanagram.jpeg){: .align-center}

## 代码实现

```java
public class ValidAnagram {
    public static boolean isAnagramMap(String s, String t) {
        Map map=new HashMap();
        int count=0;

        for(int i=0;i<s.length();i++){
            if((int)map.getOrDefault(s.charAt(i),0)<0){
                count++;
            }
            map.put(s.charAt(i),(int)map.getOrDefault(s.charAt(i),0)+1);
            if(i>=t.length()){
                return false;
            }
            if((int)map.getOrDefault(t.charAt(i),0)>0){
                count++;
            }
            map.put(t.charAt(i),(int)map.getOrDefault(t.charAt(i),0)-1);
        }

        return count==s.length()&&count==t.length();
    }
}
```