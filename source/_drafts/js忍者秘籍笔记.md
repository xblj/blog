---
title: js忍者秘籍笔记
draft: true
date: 2018-12-16 14:08:47
tags:
---

# 未来的函数：生成器和 promise

## promise

### 回调函数问题：

1. 无法使用 try-catch
2. 回调地狱
3. 多任务并发处理
   > 同时发送三个请求，必须等待三个结果都返回后才能进行下一步操作，这样就需要在每个请求回调函数中判断其他请求是否返回。
