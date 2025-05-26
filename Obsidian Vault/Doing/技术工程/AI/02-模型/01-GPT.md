
openAI 怎么强化学习

2017、Transformer
2018、2019、2020、2023 GPT-4 MOE -> 文档下载

Bert encoder
GPT decoder 强AGI方向


## 2018 GPT-01 Improving Language Understanding By G-P-T

训练分为两个阶段：
1.  第一阶段是学习一个高容量的语言模型，然后在大量的文本中进行训练
2.  接下来是微调，使用标记数据来调整模型亦适应任务


## 2019 GPT-2 Language Models are Few-Shot Learners

15B参数 48层 1600维向量

语言模型是无监督多任务学习者，数据不再是微调阶段那么重要而是在预训练阶段

某个领域专家 -> 通采
数据集变化：几乎无限量的数据，Common Crawl 无限量爬取网页数据


## 2020 GPT-3 

175B（1750亿）参数  96层 12288维向量

1.  in-context learning 上下文样本学习
2.  从预训练后，不用微调，直接去 Few-Shot 提示 prompt


## 2022 GPT-3.5 InstructGPT

监督学习又来微调GPT-3

RLHF（人类反馈强化学习）40名人员团队 ，
强化三部曲：
1. SFT 13000 -> 回答
2. Reward Model 33000  -> 4个选择
3. PPO 31000 -> 生成回答然后RM打分

prompt 测试 GPT-3 175B 和 xxx 产生两个回答，然后挑一个更好的（看起来更像强化学习）


### SFT


### RM

6B 的 RM模型，因为训练不成功 175B模型教练

输入 prompt 和 response，然后给你打分。

训练RM模型，从移除最终嵌入层的SFT模型Liner开始，替换Liner输出一个

非常难


### RL

Reinforcement Learning 强化训练，RM 模型 训练 LLM
PPO 新的方式定义总误差

策略、Action、环境变化

31000 prompt，让LLM回答，再让RM打分，RL再让分值最大化
1.  RM 很困难，分值不是很准。



## 2023 GPT-4 

MOE