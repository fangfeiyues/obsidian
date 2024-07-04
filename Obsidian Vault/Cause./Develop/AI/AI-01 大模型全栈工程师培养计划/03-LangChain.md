
LangChain 是一套面向大模型的开发框架，是AGI时代软件呢工程的一个探索和原型。
学习 LangChain 更重要的是借鉴其思想，具体接口可能很快就会发生改变。

## LangChain核心组件

### 封装模型I/O 

-  **LLMs**
  
	  大语言模型
	  ![[image-03-LangChain-20240704230916573.png]]

-  Chain Models：一般基于LLMs，但按对话结构重新封装


-  PromoteTemplate：提示词模版


-  OutputParser：解析输出




### 封装数据连接

![[image-03-LangChain-20240704232526696.png]]


-  **Document Loaders**
  
	  各种格式文件的加载器
  
  
-  **Document transfermers**
  
	  对文档的常用操作 如 spilt
  
  
-  **Text Embedding Models**
  
	  文本向量化表示，用于检索等操作？？？
  
  
-  **Verctor stores**
  
	  面向检索的向量的存储
  
  
-  **Retrievers**
  
	  向量的检索

### 封装记忆

-   Memory：历史记录，会话记录

### 架构封装

-  Chain：实现一个功能或者一系列顺序功能组合
-  Agent：根据用户输入，自动规划执行步骤，自动选择每步需要的工具，最终完成执行功能
	-  Tools：调用外部功能函数，如 google搜索
	-  Toolkits：操作某软件工具集

### Callbacks
