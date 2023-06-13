# LLMChain 和 Prompt

## Chain其他
Chain 类的入口函数，需要再详细关注一下。先回顾下，属性 `input_keys`，获取输入参数的 key 们，得到`list[str]` 。核心函数 `__call__()`，输出是一个`dict`，输入可以是一个 `dict`，也可以不是。如果不是`dict`，则用 `input_keys[0]` 作为key，输入值作为value，组成一个`dict`。两个需要记住的函数：
* `run()` 为 “单反版”，输入值如果只是value，只能有一个，如果是key-value，全部代入 `__call__()`。取返回值的第一个：`output_keys[0]`。
* `apply()` 是 “并行版”，参数是一个包含多个 `dict` 的 `list`，用每一个 `dict` 调用`__call__()` 收集所有返回值，返回一个 `list[dict]`。

所以万径归宗，最后都是调用 `__call__()`，再调`_call()`，只是对inputs和outputs，各有各的预处理和后处理。

## LLM 大语言模型
作为 LLM，基本功能就是你给他输入一个字符串，他给你续写后面的内容。输入的字符串，就是需要被续写的内容，业界叫做 Prompt（提示语）。所以 LLM 的作用，用下面这个图就可以标示清楚了，无需多说：

* Prompt -> LLM -> 续写内容

LangChain 的 LLM 基类叫 `BaseLanguageModel`，基本就是对大语言模型做了一个封装。所有 LLM 中，有续写类，补全类，翻译类等等，目前又以聊天类最为火爆，所以有一个子类叫 `BaseChatModel` 用以总结聊天类 LLM。而所有聊天类中，就数 OpenAI 的 GPT-3.5及以上系列最为火爆，其他大语言模型不堪重用。在 LangChain，OpenAI 的Chat模型类叫 `ChatOpenAI`。

用法总的来说，就是调用某个主功能函数，通常是调用 `generate_prompt()` 这个函数，在调用时，传入一个 Prompt，得到一个结果。
* Prompt -> `llm.generate_prompt()` -> `LLMResult`

这个函数叫“generate prompt”，并不是要 “产生 Prompt”，产生的不是 Prompt
，产生的是结果。这么叫的意思是说 “generate” 的参数是 `PromptValue` 类。它还有一个参数和返回值都是 `str` 的函数叫 `predict()` 、一个参数和返回值都是 `BaseMessage` 的函数叫 `generate_message()`。

而返回值类型 `LLResult` 可以认为就是一个接受 LLM API 返回的数据结构，因为我们知道 Web API 通常返回的并不是一个简单的值，而是一个 JSON 结构（的字符串）。


### 代码
我们先不要管 ChatOpenAI，我们自己来弄一个伪 LLM，你和它聊天，无论聊什么，它都回复 “你说” 加上你的话。比如：
>You：你好啊  
>AI：你说，你好啊  
>You：今天几号？  
>AI：你说，今天几号？

像 `ChatOpenAI` 继承自 `BaseChatModel` 一样，我们也继承这个类。这个类有3个接口需要实现，`_generate()` 和 它的异步版，还有一个 `_llm_type()`。
其中 `_generate()` 会收到一个 message 列表，里面的 message 会区分角色（role），角色有 System，Human（user），以及 AI（assisstant）。具体都是些什么东西，需要参考 ChatGPT API 文档，有详细说明，我就不多嘴了, [连接在这里](https://platform.openai.com/docs/guides/gpt/chat-completions-api)。我只对最后一条 Human 的 Message 做回应，如果没有HumanMessage，返回 “你说什么？”。

```python 3
from langchain.chat_models.base import BaseChatModel
from langchain.schema import ChatGeneration, ChatResult, BaseMessage, HumanMessage, AIMessage

class ChatRepeat(BaseChatModel):
	def _generate(self, messages, stop=None, run_manager=None):
		gen = '你说什么？'
		
		for msg in messages[::-1]:
			if isinstance(msg, HumanMessage):
				gen = ChatGeneration(message = AIMessage(content = f"你说，{msg.content}"))

		return ChatResult(generations = [gen])
	
	def _agenerate(self, messages, stop = None, run_manager = None):
		return self._generate(messages, stop=stop, run_manager=run_manager)
	
	@property
	def _llm_type(self) -> str:
		return "repeat-chat"

```

## LLMChain
LLMChain 是 Chain 的一个重要子类，用于将LLM封装为一个 Chain，以便和其他Chain交互。那这个 LLMChain 类，自然就需要两个重要属性：`prompt` 和 `llm`。`prompt` 代表你要给 `llm` 的提示语，`llm` 则是你要用来生成回答的模型了。

LLMChain 主要实现了这样一个功能，接收 `list[dict]`，用这些 key-value 填充 `prompt` 中的变量的 key-value。具体啥意思呢，举个例子，如果你的 Prompt 是 
`f"给我的{thing}起个名字。"`；你的 `dict` 是 `{'thing' : '猫'}`，那这个Prompt最终就是：`"给我的猫起个名字。"`。最后用这个最终 Prompt 调用 `llm.generate_prompt()`。

这个功能函数叫 `generate()`。 其他几个函数的实现和复写如下：
* `predict()` 功能和 `run` 没有区别，只是必须传入 key:value，不接受不具名参数。
* 重写了`Chain.apply()` ，复刻了 `__call__()` 的逻辑，但跳过了 `memory`，直接调用`_call()`。
* 实现了`_call()`，就是直接调用 `generate()`。


## Prompt
在使用 LLMChain 类实例的时候，输入的都是 key-value，通过 `LLMChain.prep_prompts()` 将 key-value 转换为 `list[PromptValue]`，而 `PromptValue`可以认为就是 `str`。但在创建 LLMChain 类实例的时候，需要代入一个 `BasePromptTemplate` 类。所谓 “Template”，就是这个字符串中可以含有变量，用 `"{variable}"` 这种形式。LangChain 中有一个最简单的模板Prompt类，叫做`PromptTemplate`，可以直接使用，用法如下：

```python 3
from langchain.prompts import PromptTemplate

prompt = PromptTemplate.from_template("你好，我是{my_name}。")
```
这样，我们就创建了一个带有一个变量 `my_name` 的Prompt。然后，在调用LLMChain 主要函数的时候，用 `func(my_name='xxx')` 就可以了。比如：

```python 3
# 使用刚才创建的 ChatRepeat 类 和系统自带最简单的 PromptTemplate
chn = LLMChain(llm=ChatRepeat(), prompt=prompt, verbose=True)
# 一说到语音助手，大家都会想到Javis，我这里也不免俗了。
ret = chn.predict(my_name="Javis") 
print(ret) # 你说，你好，我是Javis。
```
`predict()` 就这一种用法。如果只有一个变量，比如上面这个例子，还可以直接使用 `run()`。比如：

```python 3
ret = chn.run("Javis") # 你说，你好，我是Javis。

# 以及其他几种用法：
ret = chn.run(my_name="Javis") # 你说，你好，我是Javis。
ret = chn.run({'my_name': "Javis"}) # 你说，你好，我是Javis。
ret = chn.apply([{'my_name': "Javis"}, {'my_name': "Siri"}])
# [{'text': '你说，你好，我是Javis。'}, {'text': '你说，你好，我是Siri。'}]
```

基础的 `LLMChain` 和 Prompt 就先说到这里。后面介绍 Prompt 的高阶用法。可以这么说，玩 LLM，就是玩 Prompt。LLM 能不能按照你的心意做事，Prompt 很关键。

Lesson 3: [Prompt高阶用法](lesson3.md)

