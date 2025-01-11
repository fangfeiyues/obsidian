
## OpenAI

大模型接口很简单，像OpenAI 就只提供了 Complete 和 Embedding 两个接口，
1.  Complete 可以让模型根据你的输入自动续写
2.  Embedding 可以将你输入的文本转化成向量


### Embedding

这个 API 可以把任何你指定的一段文本，变成一个大语言模型下的向量，也就是用一组固定长度的参数来代表任何一段文本。

比如 “好评” 和 “差评” 两个字的 Embedding，对于任何一段文本评论我们也都可以通过 API 拿到他的 Embedding

