
## Pretrain 


 **前置输入**

预训练
Llama3 使用了 15t token（m 百万；b 十亿；t 万亿）

**训练目标**

	阅读大量人类文字，学习人类如何使用文字。
	标准语言建模：最大化下一个 Token 的预测概率
	自回归生成：输入为 前N 个Token，输出预测 第N+1 个Token

**训练方式**

	收集大量的Token，如GPT-3训练数据45TB大概有千亿token
	段落联合概率：一个字一个字出现概率的乘积
	LLM：算法 平衡所有 段落联合概率

**存在问题**

	预训练模型说起话来非常像接话茬，并不是在做任务（不会思考）


## Supervised Fine-Tuning 

	有监督的微调

**训练目标**

	告诉预训练后的模型，什么是对的什么事错的

**训练方式**

	使用有标签的数据进行训练，学习过程就是有监督学习；如评论 “送货真快” -> 正向标签、相似标签等
	
	有监督即有正确答案，
	自监督即自己的输出作为自己的输入，也叫自回归
	半监督：Pretrain + SFT
	
	Instruction Tuning 指令微调，告诉你指令让你干活，其实也是一种监督微调
	
	几十万条数据，大概是预训练阶段的 1/5000，需要的是数据的质量，但训练过程是类似只不过数据不一样

**存在问题**

	核心问题：一个问题对应一个优质的回答，有可能人工博士如OpenAI，有可能机器如Llama（但怎么做的更好）

## Reward Model 

奖励模型

**目标**


**训练方式**

	1、准备几万到几十万Prompt，让模型给每个 prompt 生成多个 response
	2、找一批人来做标注工作（肯尼亚2美元每天的标注工）
		1.  openAI 提供 a b c d 4个选择，进行准确性排序
		2.  meta llama 提供 a b 并通过 b 生成 偏好数据（更好数据）
	3、<Prompt、Response> pair 排序后，打分？
	核心问题：产生 <Prompt、Response> 的打分


**奖励模型 vs 监督微调**

	1. 阶段性和目的不同：SFT和奖励模型的使用处于不同的训练阶段，且目的各异
		1. SFT是在预训练之后，用于让模型适应特定任务的直接优化过程
		2. 而奖励模型则是在SFT的基础上，用于进一步提升模型输出质量的评估和优化工具
	
	2. 数据性质和使用方式不同：
		1. SFT使用的标注数据是直接的输入-输出对，模型通过学习这些明确的对应关系进行参数调整
		2. 奖励模型的数据通常是基于人类反馈或偏好比较的，它提供的是对模型输出的评价信息，而非直接的优化目标
	
	3. 优化机制不同：
		1. SFT通过监督学习直接调整模型参数，是一种直接的优化方式
		2. 奖励模型则是通过强化学习间接影响模型的优化过程，模型需要根据奖励信号进行试错学习，以找到最优的策略
	

**存在问题**

	很多专业的问题，怎么标注奖励？

## Reinforcement Learning 

强化学习

三部曲：SFT、Reward Model、强化学习

Decision Making 策略，探索 和 跟随：0 -1 的偏向值

步骤：SFT 微调模型后，根据 prompt 生成 response，然后再用 reward model 去打分，PPO 算法排分后去更新策略（反馈给SFT ）并更新 reward model 

核心问题：要强化 Reward Model，来进一步增强 SFT Model


![[4660FD53-801D-4D98-99DC-71B459ACBA53_1_102_o.jpeg]]


## LLM 训练过程

Llama-3.1 偏好数据 -> Reward Model 

### GPT


GPT 三部曲
1. Pretrain
2. SFT
3. Reward Model
4. PPO proximal policy optimization（策略）


### Llama

Llama 六轮回
1. Pretrain
2. Reward Model
3. Rejection Smaping -> 拒绝数据
4. SFT
5. DPO driect preference optimization（偏好数据再去分类好的坏的数据，继续去训练）


![[Pasted image 20250312112057.png]]

