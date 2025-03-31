## V3 技术报告
 
### V3 -  MLA

多头隐式注意力机制 Multi-head Latent Attention，解决 Q K V 缓存问题（以前 MHA Multi-head Attention）

Q K V 、相关度系数计算
[[04-Transformer#Attention]]

**缓存**

Latent KV，比如在第9个token预测第10个token的时候，前9个KV也缓存下来，Q是没必要缓存的，
同时也说明DS也没更聪明，而且占显存原因是每层都要自己缓存自己，
96头 * 128KV向量 * 96层 * 1300个向量 = 30亿 * 4 = 120个字节 / 1024 KB / 1024 MB / 1024 = 11G 显存

一次问答，600多个汉字，1300多个token，1300多个向量。然后再开始回答问题，上下文会越来越大，现在很多模型有20w token 接近10w汉字... 则缓存越来越多


**怎么减少缓存**

低秩联合压缩（如把海绵里的水压缩些）最终把 1300维向量压缩成 100


### V3 - DeepSeek MOE

混合专家 Mixture of Experts，来替代 FFN（Feed Forward）

分而治之策略的神经网络架构，它将复杂的问题分解为多个子问题，每个子问题由一个独立的模型（称为专家）进行处理


## R1 技术报告

























