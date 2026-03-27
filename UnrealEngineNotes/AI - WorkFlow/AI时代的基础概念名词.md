原视频：[从 LLM 到 Agent Skill，一期视频带你打通底层逻辑！_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1E7wtzaEdq/?spm_id_from=333.1387.favlist.content.click&vd_source=2d5f5b0c789f1187890235f6fa20045c)

AI时代来临互联网上充斥着很多新名词。

![[AI - WorkFlow/AI时代的基础概念名词Media/1.png]]

# LLM
LLM是Large Language Model（大语言模型）的意思。LLM基本上是由google团队在2017年提出的Transform架构，随后被OpenAI推向全世界从而演变而来的。大模型可以理解为我们今天的chatgpt，豆包，元宝这些东西。

# Token
Token通常是**令牌**的意思在如今的AI时代代表**词元**可能来说更准确些。我们在客户端输出一串文本问题，会先进入Tokenizer流程，这个流程中有编码和解码两部分。
## 编码
这串文本会先被切分转化成Token也就是文本被切分成的最小片段。一个Token会被映射到一个数字上去，这个数字被称为TokenID。
![[AI - WorkFlow/AI时代的基础概念名词Media/2.png]]