原视频：[从 LLM 到 Agent Skill，一期视频带你打通底层逻辑！_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1E7wtzaEdq/?spm_id_from=333.1387.favlist.content.click&vd_source=2d5f5b0c789f1187890235f6fa20045c)

AI时代来临互联网上充斥着很多新名词。

![[AI - WorkFlow/AI时代的基础概念名词Media/1.png]]

# LLM
LLM是Large Language Model（大语言模型）的意思。LLM基本上是由google团队在2017年提出的Transform架构，随后被OpenAI推向全世界从而演变而来的。大模型可以理解为chatgpt，deepseek这些东西。

# Token
Token通常是**令牌**的意思在如今的AI时代代表**词元**可能来说更准确些。我们在客户端输出一串文本问题，会先进入Tokenizer流程，这个流程中有编码和解码两部分。
### 编码
这串文本会先被切分转化成Token也就是文本被切分成的最小片段。一个Token会被映射到一个数字上去，这个数字被称为TokenID。随后这串TokenID会传入大模型。
![[AI - WorkFlow/AI时代的基础概念名词Media/2.png]]

### 解码
经过大模型的处理后会返回一个TokenId。然后Tokenizer会通过解码把这个TokenID再映射成Token，需要注意的是解码环节是
不需要切分的，因为模型每次只会给出一个TokenID。如果模型的话没有说完就继续吐第二个，第三个token...
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/3.png]]

## 
所以Token是大模型处理文本的最基本单位。Token并不一定是按我们理解上的词来划分的。
OpenAI提供了一个页面可以把字符串转化成Token。这里工作坊被拆成了两个词而不是一个词。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/4.png]]

英文中常见单词是一个token，但也有例外如helpful被拆成两个token。还有些特殊字符被拆成3个token来使用。平均来看1个token等于0.75个英文单词，1.5~2个汉字。

# Context和ContextWindow
大模型在和我们沟通的时候，如何记住之前的聊天内容？我们每次在给大模型发送消息的时候，不只是发送单条问题，程序会把当前问题连同之前的内容一起发送给大模型。这就是Context上下文（大模型每次处理任务接受到的信息总和）问题和对话历史都是Context的一部分。除此以外还有别的内容比如大模型正在输出的每个token也会被追加进来，工具列表，System Prompt。Context从某种程度上来看就是大模型的临时记忆体。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/5.png]]

Context能有多大能塞多少Token呢？这就引出了Context Window的概念译为上下文窗口的概念。Context Window代表Context能容纳的最大Token数量。目前主流的大模型的Context Window都来到100万级别的大小了。通常能容纳150万个汉字，相当于整个哈里波特全集的内容了。
# RAG
假如公司有一个很长的（上千页的）产品手册，我们希望公司根据产品手册的内容回答用户的问题。不可能把产品手册作为Context Window发送给大模型，即使模型的Context Window
不被撑爆我们的成本也会变得很大。RAG技术可以抽取用户手册中和用户问题最为匹配的片段，只把这些片段发送给大模型。让大模型只根据这几个片段来回答用户问题。这样大模型接收到的就不是一整本书了只是几段话。这样就不受Context Window限制了成本也会低很多。

# Prompt
中文译为提示词，是大模型接收的具体提问或指令，Prompt不是什么特别复杂高端的东西。比如我们向大模型提需求：
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/6.png]]

但是Prompt如果是是如上，大模型可能会给你一首古诗，现代诗或者打油诗。因为我们的Prompt太模糊了。Prompt怎么写直接决定了大模型的输出质量。一个好的Prompt应该是清晰的，具体的，明确的。比如我们可以如下输入Prompt，得到的回答就更清晰明确了。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/7.png]]

这就是为什么有个专门的领域叫Prompt Engineering（就是研究怎么把话说清楚让大模型更明白你的意图）。这个领域曾经比较火现在还在提他的人寥寥无几，因为大模型越来越强了即使我们现在的Prompt不太明确，它也能懂我们的意图。

# User Prompt和System Prompt
我们不仅要告诉大模型要处理的任务，还要告诉大模型它的人设和做事规则。说明具体任务的就是User Prompt是用户输入的，说明人设和做事规则的是System Prompt是开发者在后台配置的。

一个例子：假设要做一个数学辅导机器人 ，我们希望他不要直接告诉学生答案而是要引导学生思考。这时候就需要两种Prompt。
1. System Prompt：我们在后台设置，用户看不到。
2. User Prompt：用户输入的问题。
没有System Prompt的情况下我们大模型可能会直接给我们返回8。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/8.png]]

# Tool
大模型的弱点，无法感知外界环境，如下面这样的一段对话：
大模型的能力是通过训练数据来推测下一个词，无法实时去天气预报网站获取最新数据。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/9.png]]

Tool中文译为工具它的本质上是一个函数。我们传入输入Tool就返回输出。
有了它大模型就可以回答天气相关的问题了。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/10.png]]

我们可以看下整体流程。这里会涉及到一个新的东西叫**平台**
因为用户，大模型，天气查询工具这三个角色没办法直接对话。所以我们需要平台这个角色来做传递信息的工作。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/11.png]]

在这个过程中每个角色都有自己的职责：
- 大模型：选择工具并生成对应的工具参数，拿到工具的结果后模型需要对结果进行归纳总结。注意：有很多初学者认为模型可以直接调用工具，这个事情应该是由平台来负责完成。
- 工具：查询天气。本质是提供大模型感知外部环境的能力。
- 平台：串联整个流程。

# MCP
在上面的流程中平台要把工具列表传给大模型，还要能调用工具。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/12.png]]

我们首先做的是要把工具接入到平台中。这样平台才知道可用工具列表，以及每个工具的用途，参数和调用方法等等。但是这套接入的规范每个平台都不一样。如果用的chatgpt就要使用OpenAI的接入规范，如果用的Claude就要按照Anthropic的规范，如果是用Gemini就要用Google的规范。接入是需要写接入代码的。同一个工具可能要写3遍，所以有人就提出能不能有个通用的方案，让所有的平台都遵循这一个标准这样工具的开发者只用写一套代码。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/13.png]]

MCP（Model Context Protool：模型上下文协议）就是这个统一的接入规范。有了MCP后工具的开发者只要按照MCP的规范开发一次工具就可以被所有支持MCP的平台使用了。这就像是所有手机都用Type-C接口一样，有了统一标准大家都会方便很多。

# Agent
现在我们已知大模型能借助工具感知外部世界。工具又可以用MCP这种方式统一接入。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/14.png]]

按理来说已经很强了，可是还差点东西。假如用户问如下问题。整体调用流程如下：
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/15.png]]

从某种程度上来说大模型已经有了一定的自主规划能力。我们称这种自主规划，自主调用工具，直至完成用户任务的系统为Agent。现在市面上有很多Agent产品，使用的构建模式也各不相同。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/16.png]]

# Agent Skill
虽然Agent已经很强大了，但是在实际的高频使用中会遇到一个新的痛点。
当我们把大模型作为我们的出门小助手，我们希望有自己的一套出门习惯，和希望的输出格式。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/17.png]]

在没有额外设定的情况下，我们提出如下问题：大概率会得到一堆废话。最终的输出格式也无法满足我们的要求。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/18.png]]

为了拿到满意的结果我们需要每次在prompt中输入格式要求。会变成如下：
如果每次我们都要输入这么一大段是不是有点太反人类了。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/19.png]]

这个时候就需要Agent Skill登场了。Agent Skill本质上是给Agent看的说明文档，比如刚才的那个出门场景我们就可以写成如下的Agent Skill本质上就是一段markdown文档。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/20.png]]

这个Agent Skill整体结构可以分为两部分：
上面这块叫做元数据层，相当于这份说明文档的封面，告诉Agent这个技能叫什么。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/21.png]]

下面这部分被叫做指令层
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/22.png]]

上面提到的输出规则。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/23.png]]

定义好Agent Skill后我们要把他存到硬盘里面指定的地方
拿Claude举例mkdir后面的目录名必须和skill同名。
vim SKILL.md 这个md文件的名字是硬性规定。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/24.png]]

然后把Agent Skill中的内容粘贴过来
wq保存退出
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/25.png]]

随便找一个空的文件夹启动Claude Code
在启动的过程中Claude就会发现Skill文件夹中多了个名为go-out-checklist的Agent Skill
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/26.png]]

它就会读取对应的SKILL.md文件中的元数据，下面的指令层内容暂时不读取。只有在用户提出的问题和元数据层的描述相关时才读取。

下面的Agent Skill明确需要定位和天气这两个工具
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/27.png]]

需要Claude Code中有这个些MCP工具，运行指令可以验证。
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/28.png]]

输入问题
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/29.png]]

首先会请求调用定位工具，我们同意一下
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/30.png]]

同样同意调用天气工具
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/31.png]]

最后拿到所有信息后输入规定格式的答案
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/32.png]]

Agent Skill 还有很多高级功能比如运行代码 引用资源等等。它的渐进式披露机制也是一大特色，可以节省很多的token。

# 总结
![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/33.png]]

![[UnrealEngineNotes/AI - WorkFlow/AI时代的基础概念名词Media/34.png]]