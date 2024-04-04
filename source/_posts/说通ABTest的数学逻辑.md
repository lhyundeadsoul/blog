---
title: 说通ABTest的数学逻辑
date: 2020-10-27 11:24:23
tags:
  - 测试
  - 大数据
  - 方法论
---
A/B实验从本质上来说是一个基于统计的假设检验过程；
- A/B实验从本质上来说是一个基于统计的假设检验过程
    
- 好的表达：实验组转化率比对照组高了1% -> 我认为实验组转化率相比对照组转化率高0.8-1.2%（1% ± 0.2 %）, 置信度为95%
    
- 正态分布的数据特点：对于正态分布的数据，有68.2%个落在距离总体均值1个标准差 (σ) 的范围内，95.4%个落在距离总体均值2个标准差 (σ) 的范围内，99.7%个落在距离总体均值3个标准差 (σ) 的范围内(中心极限定理和正态分布的运用)，有95%个落在距离总体均值1.96倍个标准差 (σ) 的范围内。
    
- 中心极限定理 (Central Limit Theorem): N次抽样，形成N个样本均值，这N个样本均值的分布趋向于服从正态分布。
    
- 置信区间的上界是样本均值+抽样误差，下界是样本均值-抽样误差，95%置信度下的抽样误差是1.96*样本标准差
    
- 优先支持我们预期结论的反面观点（原假设），再证伪这个观点，以此证明预期观点（备择假设）。这种方法可以避免“确认偏误”（Confirmation Bias）的出现，即选择性的找支持自己观点的证据而忽视反对的证据的现象。
![abtest](https://mmbiz.qpic.cn/sz_mmbiz_jpg/X5cB3w6FrP6SUKsgm7qcghdicNt0bSU4O9Qib9CBica4Ofbhk2Rw5U5tQia1NscRe1aSxlCMuYKEk0p8GqPrpibH2IQ/0?wx_fmt=jpeg)
![abTest](/images/abtest.jpeg)


https://mp.weixin.qq.com/s?__biz=MzU1ODEzNjI2NA==&mid=2247487223&amp;idx=1&amp;sn=306598fda6c931e7fbf023cf07959f20&source=41#wechat_redirect
https://mp.weixin.qq.com/s?__biz=MzU1ODEzNjI2NA==&mid=2247487225&amp;idx=1&amp;sn=3598f1caf4d3fd5c4a77962a9f86f692&source=41#wechat_redirect
