
## 1、Pretrain 

k 千、m 百万、b 十亿、t 万亿
如 Llama3 使用了 15t token 参数（几乎包含了所有的文字数据）

训练目标

阅读大量人类文字，学习人类如何使用文字
标准语言建模：最大化下一个 Token 的预测概率
自回归生成：输入为 前N 个Token，输出预测 第N+1 个Token


### 训练方式

收集大量的Token，如 GPT-3 训练数据 45TB 大概有 千亿token

段落联合概率：让一个模型，能一字不落的生成整个段落的概率即一个字一个字出现概率的乘积
如买7位数的彩色球，每个球都是一个概率，7/10 * 6/10 ...（无顺序的）

假设2000个汉字是一篇文章，7.5万亿就会有37.5亿个段落，这么多段落怎么平衡LLM综合不满意度？
不断的升级版本，怎么综合概率？

Llama-2 65B 1.4万亿token 2048个A100 训练21天

### Q&A

预训练模型说起话来非常像接话茬，并不是在做任务？如今天天气怎么样、这家公司怎么样等


## 2、SFT

Supervised Fine-Tuning 自监督微调，通过高质量的人工标注数据让模型“说人话”

训练目标：告诉预训练后的模型怎么做任务，什么是对的什么事错的什么是相似的。

	1.  有监督的微调，有标注标签的数据进行训练（标签就是正确答案） 如评论 “送货真快” -> 正向标签
	2.  无监督的微调，
	3.  自监督(自回归)，自己给自己打标签，如预训练阶段，输入和输出在不断变换如第4个汉字输出后又变成输入
	4.  半监督，Pretrain + SFT

### 训练方式

Instruction Tuning 指令微调，告诉你指令让你干活，其实也是一种监督微调。如 QianWen-72B-Instruction

SFT阶段需要的不多，大概是几十万条数据，大概是预训练阶段的 1/5000
需要的是数据的质量，但训练过程是类似，只不过数据不一样，人工费可能5000w美刀

80个博士在不同方向一问一答，GPT4写了几万条。准备的什么数据，就会做什么任务。如做作业、改作业

**损失函数 L = − ∑​ yi​ log(y^​i​)**

其中：
- yi​ = 真实标注的目标 token
- y^​i​ = 模型在当前位置生成的 token 的概率分布

### Q&A

- SFT 数据量比 Pretrain 更少但却能提升输出准确性？（预训练是学习语言和知识，SFT是调整说话方式）

	1.  增强数据的多样性提升 SFT 数据集的覆盖度
	2.  指令微调，不需要模型完全“学习”每种任务，而是通过“理解指令”生成更泛化的结果，通过小样本学习（Few-shot）提升泛化能力
	3.   RLHF（强化学习人类反馈）
		- 通过奖励模型（Reward Model）来衡量模型输出的好坏
		- 使用策略优化（PPO 等）让模型朝着更优方向生成答案
		- RLHF 会显著减少幻觉（Hallucination）和偏差（Bias）

- SFT 是怎么让 LLM 得算法结果更倾向于 ta？
- 会做不行，怎么做的更好 -- Reward Model?

## 3、RM

奖励模型 Reward Model，其实也是一个概率模型，区别于 LLM 是一个单独模型。

### 训练方式

1、准备几万到几十万Prompt问题，让LLM模型给每个 prompt 生成多个 response 即 <Prompt, List<Response,>>

2、找一批人来做标注工作（肯尼亚2美元每天的标注工）
1.  openAI 对一个 Prompt 回答 a b c d 4个选择，进行准确性排序
2.  meta llama 提供 a b 并通过 b 生成 偏好数据（更好数据）
   
3、<Prompt、Response> pair 排序后，怎么打分？完生成偏好数据 Reference Data

4、训练 Reward Model，其实就是再训练SFT模型


### Q&A

- 很多专业的问题，怎么标注奖励？
- 为什么SFT后的模型会产生多个答案？多段落概率来看不是越高越出答案？
  
-  奖励模型 vs 监督微调？
  

## 4、RL

Reinforcement Learning 强化学习
RLHF （Reinforcement Learning Human Feedback） 基于人类反馈的强化学习，人类反馈即上面说的标注的偏好数据

Decision Making 策略，选择 新的探索 还是 老的跟随：0 -1 的偏向值

### 步骤

1、SFT 微调模型后，根据 prompt 生成 response

2、然后再用 Reward Model 去打分，

3、PPO 算法排分后去微调 LLM 模型，并更新 Reward Model

4、然后不停的循环，同时 RM 也要不断更新训练（RM就是教练的角色，当你更牛的时候教练也要更厉害）


类似于游戏的NPC，每个动作后都可以得到回馈。

![[Pasted image 20250318111927.png]]

	1. Step1：生成有监督的数据
	2. Step2：训练奖励模型的ABCD答案
	3. Step3：LLM 生成 response 后，RM 对 response 打分后，去升级LLM算法


### Q&A

要强化 Reward Model，来进一步增强 SFT Model


## -> LLM 训练过程

Llama-3.1 偏好数据 -> Reward Model 

### GPT

GPT 三部曲
1. Pretrain
2. SFT
3. Reward Model
4. PPO proximal policy optimization（具体的模型迭代方式？包括损失函数等）

### Llama

Llama 3.1 六轮标注偏好数据
1. Pretrain
2. Reward Model
3. Rejection Smaping -> 拒绝数据
4. SFT（数据不同于人工生成的，机器生成的，同时模型过滤下Data）
5. DPO driect preference optimization
   （偏好数据不同于SFT都是优质数据，它会再去分类好的坏的，继续去训练）

![[Pasted image 20250312112057.png]]


## Next

下一步要解决的问题就是，怎么用 Pretrain 和 SFT 的数据训练 以及 LLM 的 DPO算法，怎么 输出最终的 Response