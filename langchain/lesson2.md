# LLM 和 Prompt

## LLM 大语言模型

作为 LLM 大语言模型，基本功能就是你给他输入一个字符串，他给你续写后面的内容。输入的字符串，就是需要被续写的内容，业界叫做 Prompt（提示语）。目前大语言模型界的扛把子是 OpenAI 的 GPT系列。GPT 有两种，Completion 和 Chat。Completion 是单纯的文字续写，比如 GPT-3。而 Chat 是基于 Completion 的对话形式，比如 GPT-3.5（ChatGPT）和 GPT-4（虽然也是 Chat 类，但它不叫 ChatGPT）。这两种模型的 Prompt 有一些区别。Completion 的 Prompt 就是一段文字，然后它来续写。而 Chat 因为要区分聊天内容是谁的，所以 Chat 的 Prompt 有3种：System, Assisstant, User。System 用来介绍对话的前提情况，背景设定等。详细可以参考 [OpenAI 的 API 文档](https://beta.openai.com/docs/guides/gpt/chat-completions-api)。

所以 LLM 的作用，用下面这个图就可以标示清楚了，无需多说：

* Prompt -> LLM -> 续写/回答

LangChain 的 LLM 基类叫 `BaseLanguageModel`，基本就是对大语言模型做了一个封装。所有 LLM 中，有续写类，补全类，翻译类等等，目前又以聊天类最为火爆，有一个子类叫 `BaseChatModel` 用以总结聊天类 LLM。在 LangChain，OpenAI 的 Chat 模型类叫 `ChatOpenAI`。

`BaseLanguageModel` 这个类，用法总的来说，就是调用某个主功能函数，如果调用 `predict()` 这个函数，在调用时，传入一个 Prompt，得到一个结果。
* Prompt -> `llm.predict()` -> 输出文字


### 代码
我们先不要管 ChatOpenAI，我们自己来弄一个伪 ChatLLM，你和它聊天，无论聊什么，它都回复 “你说” 加上你的话。比如：
>You：你好啊  
>AI：你说，你好啊  
>You：今天几号？  
>AI：你说，今天几号？

`ChatOpenAI` 继承自 `BaseChatModel`，我们也继承这个类。这个类有3个接口需要实现，`_generate()` 和 它的异步版，还有一个 `_llm_type()`。
其中 `_generate()` 会收到一个 message 列表，里面的 message 会区分角色（role），有 System，Human（user），以及 AI（assisstant）。具体都是些什么东西，需要参考 ChatGPT API 文档，有详细说明，我就不多嘴了。我只对最后一条 Human 的 Message 做回应，如果没有HumanMessage，返回 “你说什么？”。

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

cr = ChatRepeat()
prompt = "Hello world!"
ret = cr.predict(prompt)
print(ret) # 你说，Hello world!
```


## Prompt 提示语类

Prompt 如果只用字符串，不高级。LLM 的返回值，也不是简单的字符串。LangChain 中有两个类来表示 Prompt：
* `BaseMessage`, 其实就是 `str` 的极简包装。 
* `PromptValue` 可以用 `to_string()` 得到 `str`, 或 `to_messages()` 得到 `list[BaseMessage]`。

> 助记：`PromptValue` 就是 `BaseMessage` 就是 `str`。以后只要说 “`BaseMessage`”, 那不管实际是 `PromptValue` 还是 `str`，都一回事。

之前有说过，LLM有两种类型，Completion 和 Chat。`BaseChatModel` 对应 Chat。`BaseLLM` 则对应 Completion。Chat 是流行趋势，`BaseLLM` 我就不做介绍了，以后只对 Chat 类 `BaseChatModel` 做介绍。（不过 OpenAI 的 GPT-3 虽然质量不如 GPT-3.5，但是目前相较而言，接口访问速度快，访问限制少，有时候也是一个不错的平替）

之前有提到 `BaseChatModel` 这个类的主要工作流程是：

* Prompt (`str`) -> `llm.predict()` -> `str`

它的基类 `BaseLanguageModel` 有三个同功能 __虚函数__，除了 `predict()` 外，还有两个：
* `predict_messages()` 接受 `list[BaseMessage]` 返回 `BaseMessage`
* `generate_prompt()` 接受 `list[PromptValue]` 返回 `LLResult`

返回值类型 `LLResult` 可以认为就是一个接受 LLM API 返回的数据结构，因为我们知道 Web API 通常返回的并不是一个简单的值，而是一个 JSON 结构（的字符串）。

而 `BaseChatModel` 自己又添加了一个函数：
* `generate()` 接受 `list[list[BaseMessage]]` 返回 `LLResult`

说回到 `BaseMessage` 和 `PromptValue`。

`BaseMessage` 有这些子类：
* `SystemMessage`， 代表系统 Prompt
* `HumanMessage`，代表用户输入
* `AIMessage`，代表AI回答
* `ChatMessage`， 可以自己指定 role 和 content

而 `PromptValue` 有这么个子类：
* `ChatPromptValue` 包含 `list[BaseMessage]`。

我们来看看我们的 ChatRepeat 类，使用其他几个函数，效果如何。

```python 3
# ChatRepeat 类 代码在上一篇

from langchain.schema import ChatMessage
from langchain.prompts.chat import ChatPromptValue

cr = ChatRepeat()

ret = cr.predict('Hello World!')
print(ret) # 你说，Hello World!

messages = [ChatMessage(role='human', content="非HumanMessage被ChatRepeat忽略"), HumanMessage(content="你好吗？")]

ret = cr.generate(messages=[messages])
print(ret) # generations=[[ChatGeneration(text='你说，你好吗？', generation_info=None, message=AIMessage(content='你说，你好吗？', additional_kwargs={}, example=False))]] llm_output={}

ret = cr.predict_messages(messages=messages)
print(ret) # content='你说，你好吗？' additional_kwargs={} example=False

ret = cr.generate_prompt([ChatPromptValue(messages=messages)])
print(ret) # generations=[[ChatGeneration(text='你说，你好吗？', generation_info=None, message=AIMessage(content='你说，你好吗？', additional_kwargs={}, example=False))]] llm_output={}

```

我们再用 ChatGPT 实际感受一下：

```python 3
from langchain.chat_models import ChatOpenAI
from langchain.schema import HumanMessage

os.environ["OPENAI_API_KEY"] = "sk-your-key"

llm = ChatOpenAI(verbose=True)
messages = [HumanMessage(content="你好吗？")]
ret = llm.generate(messages=[messages])
print(ret) # generations=[[ChatGeneration(text='我很好，谢谢。作为一个AI助手，我没有情绪，但我很乐意帮助你。有什么我可以帮你的吗？', generation_info=None, message=AIMessage(content='我很好，谢谢。作为一个AI助手，我没有情绪，但我很乐意帮助你。有什么我可以帮你的吗？', additional_kwargs={}, example=False))]] llm_output={'token_usage': {'prompt_tokens': 13, 'completion_tokens': 50, 'total_tokens': 63}, 'model_name': 'gpt-3.5-turbo'}
```


语言模型的 Prompt 是直白的，就是 `str` 或 `list[str]`，没有什么花花肠子，但这不够灵活。不能动态改变内容。所以就有了 Prompt 模板。

Lesson 3: [Prompt高阶用法](lesson3.md)
