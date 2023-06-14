# Prompt模板 及 LLMChain

## Prompt 模板

`BasePromptTemplate` 类让 Prompt 可以处理字符串中的变量，让字符串变成模板。功能类似 python 的 f-string。比如这样：

```python 3
thing = "猫"
prompt = f"给我的{thing}起个名字。"
```
这个基类里面，有关于 Prompt 中变量的信息。它的子类孙类们，就是 `XxxPromptTeplate`，里面又各自包含了n个 “`BaseMessage`”用于处理，主要有这三种函数，获得给模板赋值后的 Prompt：
* `format() -> str`
* `format_prompt() -> PromptValue`
* `format_messages() -> List[BaseMessage]`

比如有一个孙类 `PromptTemplate`，里面 `format_prompt()` 函数可以认为就是直接用 python 的 `str.format()` 来赋值字符串中的变量 。

使用方法直接上代码：

```python 3
from langchain.prompts.prompt import PromptTemplate

pt = PromptTemplate.from_template("你好我是{my_name}。")
# 这里给 Prompt 里的变量赋值
pv = pt.format_prompt(my_name='Javis')
# 这个时候内容就被替换成 "你好我是Javis。" 了。
ret = cr.predict_messages(messages=pv.to_messages())
print(ret) # content='你说，你好我是Javis。' additional_kwargs={} example=False
```
这样，模板的功能就实现了。如果就一条消息，这样做很好用。但是作为 Chat 模型，你不能只发送 Human 的第一句话。AI的回答，后续历史对话记录，你也要发送出去。类似这样：

```python 3
chat_history = [
	SystemMessage(content="你是{my_name}，一个得力的助手。"),
	HumanMessage(content="你好吗？"),
	AIMessage(content="我很好。"),
	HumanMessage(content="你叫什么名字？"),
	AIMessage(content="我叫Javis。"),
	HumanMessage(content="你能做什么？"),
]

messages = [format_message(msg, my_name="Javis") for msg in chat_history]

ret = cr.predict_message(messages=messages)
```

这个时候， `ChatPromptTemplate` 出场了。

## ChatPromptTemplate

刚才说了，需要一个 `BaseMessage` 列表，来记录对话历史。但是 `BaseMessage` 没有模板功能。但是，可以用一个循环，对所有 `BaseMessage` 做一遍 `format_prompt()` 就好了。`ChatPromptTemplate` 差不多正是这样做的。

差不多是什么意思？如果是一个 `BaseMessage` 列表，`ChatPromptTemplate` 并不处理，LangChain 又另外搞了一个类，叫 `BaseMessagePromptTemplate`，用来表明这是一个带有 Template 的 `BaseMessage`。

但是，带有 Template 的 `BaseMessage` 不就是 `BasePromptTemplate` 本身吗？`ChatPromptTemplate` 就是 `BasePromptTemplate` 的子类。如果 `ChatPromptTemplate` 想包含多个带有 Template 的 `BaseMessage`，完全可以包含一个 `list[BasePromptTemplate]` 就好了呀？

个人认为，另建一个简单的 `BaseMessagePromptTemplate` 类，一是为了简化逻辑，二是 `BasePromptTemplate` 内部变量逻辑还是有些多，代码有些重。

## 捋一遍

1. 如果是对话记录，又没有变量，就是用 `list[BaseMessage]` 直接喂给 llm。  
2. 如果是简单的 Prompt 模板（Prompt 中有变量），就直接用 `str` 构建 `PromptTemplate` 类，转 (4.)。
3. 如果是对话记录，且有变量，就用 `list[ChatPromptTemplate]` 来创建 `ChatPromptTemplate` 类 (4.)。
4. 然后调用 Prompt 模板的 `format_prompt()` 就可以得到 `list[BaseMessage]`，就可以喂给 lllm 了。
5. 当然，第一种情况，也可以用 `list[BaseMessage]` 来构建 `ChatPromptTemplate` 类，结果是一样的。

```python 3
from langchain.prompts.chat import SystemMessagePromptTemplate, HumanMessagePromptTemplate

messages = [
	SystemMessagePromptTemplate.from_template("Your name is {ai_name}."),
	SystemMessagePromptTemplate.from_template("Your are very helpful."),
	HumanMessagePromptTemplate.from_template("Give a name to my {product}."),
]

pt = ChatPromptTemplate(input_variables=['ai_name', 'product'], messages=messages)

prompts = pt.format_prompt(ai_name = 'Javis', product="book")
ret = cr.generate_prompt(prompts=[prompts])
print(ret)
# generations=[
#   [
#       ChatGeneration(
#           text='你说，Give a name to my book.', 
#           generation_info=None,
#           message=AIMessage(content='你说，Give a name to my book.', 
#               additional_kwargs={}, 
#               example=False)
#       )
#   ]
# ] 
# llm_output={}
```



