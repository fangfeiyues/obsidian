
## Plugins

GPT提供了很多插件能力，并制定标准的接口调用模版 

## Function Calling

简单来说，就是工具使用

自主解析内容并结构化填槽，自主决策使用工具并结构化调用
1. System Prompt 写明都有哪些工具可调用
2.  意图识别
3.  决策是否使用工具
4.  调取工具设定的信息传递结构
5.  调取所有上下文内容解析并填槽
6.  缺少信息就发起追问，直到全部填完

	![[Pasted image 20250407194251.png|500]]


## RAG

	Retrieval-Augmented Generation 私有知识库

将参考资料、样例放在prompt中，就叫 In-Context-Learning
但上下文窗口宽度有限，所以需要一个知识库，需要在知识库里找到一些有用的信息

1、构建可检索的知识库

	1. 知识整理
	2. 数据清理及格式化
	3. 内容切分：较小的知识片段Chunk
	4. 向量化：将每个知识片段转化为向量表示，如OpenAI 的 Embedding
	5. 关联元数据
	6. 载入向量数据库，并建立索引：如FAISS、Pinecone等
	7. 部署集成

2、模型调用知识库完成用户任务

	1. System Prompt 写明哪些情况调用知识库
	2. 模型解析用户意图
	3. 将 User Prompt 转成向量，去向量数据库比较相似度
	4. 选取相似高的一条或多条知识片段，并取出知识片段
	5. 将检索出的知识片段与原prompt合并一起组成新的prompt

RAG & LLM
	![[Pasted image 20250407201733.png|500]]



提高检索准确率
1. 知识片段有明确主题、多颗粒混合切分
2.  知识片段的上下文相邻片段也可以一并取出
3.  检索多召回一些结果，再用单独的模型去做重排序
4.  向量 + 关键词混合检索


表格处理（文件内容处理的时候会被处理成纯文字，而丢失表格结构）
1.  GPT-4o给图片写一段文字
2.  这段文字生成向量，加入向量数据库
3.  如果向量被检索到，再取出这张图片
4.  将图片作为参考资料，跟所有 Prompt 一起给到模型


多样使用方法



## MCP

- **MCP**

	![[Pasted image 20250430173234.png|500]]
	
	1.  MCP Host：主机，启动连接的LLM，并集成 MCP IDE 如 Claude Desktop
	2.  MCP Client：复杂请求 Server
	3.  MCP Server：提供上下文与三方连接，然后暴露数据与功能给 Client 使用
	4.  Transport：Client 与 Server 之间协议如标准输入/输出stdio、服务器发送事件SSE等

 
-  **MCP  vs  Function Call**

	![[Pasted image 20250430174346.png|500]]
	
	Function Call 是大语言模型LLM与外部工具或API交互的核心机制，是一个基础能力，识别什么时候要工具，可能需要什么类型的工具的能力
	
	MCP则是工具分类的箱子，而非取代Function Call，而是在其基础上，联合Agent一起完成任务


-  **MCP乱局**
  
	1.  开发的难题
	   
			MCP早期对于云端服务器 remote 采用了一种双链接机制，
			一条是SSE的长链接来单向的从服务端到客户端推送消息，还有一条是发消息的http常规请求短链接
			这样对于云端接入服务端，MCP接口意味着复杂的成本
		
	2.  市场的混乱




- 「MCP入门」 https://mp.weixin.qq.com/s/KOXJO_T8V1ebKE_Rbu6Mpw
- 「MCP很好，但不是万灵药」 https://mp.weixin.qq.com/s/19NAsU0nYeEC822U2LttUw



