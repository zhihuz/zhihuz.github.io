---
layout: post
title: "Count and Say"
tags: [LintCode, python,count and say,解题报告]
---
##Description
The count-and-say sequence is the sequence of integers beginning as follows:

1, 11, 21, 1211, 111221, ...

1 is read off as "one 1" or 11.

11 is read off as "two 1s" or 21.

21 is read off as "one 2, then one 1" or 1211.

Given an integer n, generate the nth sequence.

Notice

The sequence of integers will be represented as a string.

Example

Given n = 5, return "111221".
##Solution
{% highlight python %}

class Solution:
    # @param {int} n the nth
    # @return {string} the nth sequence
    def countAndSay(self, n):
        # Write your code here
        if n ==1:
            return '1'
        a = '1'
        b = ''
        for i in range(1,n):
            s = a[0]
            count = 0
            for j in a:
                if s == j:
                    count +=1
                else:
                    b += str(count)+s
                    s = j
                    count = 1
            b += str(count) + s
            a = b
            b = ''
        return a

{% endhighlight %}
