
1.  用自然语言连接系统的认知，面向未来思考系统间的集成问题
2.  了解 OpenAI Plugins 的基本原理和市场表现，对行业格局产生一些感知
3.  学会 OpenAI Function Calling 的基本用法 

## Plugins

学习Plugins前，先了解 ChatGPT 及所有大模型都有的两大缺陷：
1.  没有最新消息。大模型训练周期很长，且更新一次耗资巨大
2.  没有真逻辑，它表现出的逻辑、推理，是训练文本的结果

### 开发一个 Plugins

1.  定义一个Description文件
2.  定义一个Yaml文件，描述接口信息

### 失败的原因

它暂时歇菜了..

1.  缺少「强Agent」调度，只能手工选择三个 plugins，使用成本太高 -- 解决此问题相当于 App Store + Siri
2.  不在「场景」中，不能提供端到端一揽子服务 -- 解决此问题，就是全能私人助理？
3.  延迟 -- 至少两次 GPT 生成 + 一次 WEB API调用


## Function Calling


