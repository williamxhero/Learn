# 从头说起

Lanchain的结构设计非常精妙，分析过源码后，受益匪浅。我在这里尝试从头开始，记录下Lanchain的设计思路和用法，希望能达到知其然并知其所以然的效果，在以后使用的时候，可以更加得心应手。

说道langchain，就需要从2个核心类说起：Chain 和 Memory

## Chain
Chain 类是整个框架的核心，从库名中也可以看得出来。Chain 有一个主要入口函数，`__call__()`，这是一个 python 特殊函数，就是你可以直接把类实例当函数用，比如：

```python
chain_instant = Chain(...)

outputs = chain_instant(inputs)  # 此处调用的就是 __call__(...)
```
inputs 和 outputs 都是`dict`类型，就是一堆 key-value。具体是什么key，什么value，这个由子类决定和使用，Chain作为父类，并不关心具体含义和数值。

还有几个其他入口函数，最终也是调用了 `__call__()`。此处先不做介绍，我们保持简单和关键。

Chain 里有一个 memory 变量，memory变量主要作用是保存上次调用 `__call__()` 的返回值，在下次调用的时候使用。

`__call__()` 函数主要做了这么3件事：
1. 预备 inputs：检查 inputs 合法性。
2. 用 inputs 调用 `_call()` 函数：这是一个虚函数，子类必须实现，返回 `dict`作为 outputs。
3. 预备 outputs：检查 outputs 合法性，将 inputs 和 outputs 合并成一个 `dict`，返回出去。

如果没有设定memory，则1.和3.都不执行，等于是只调用`_call()`函数和检查输入输出合法性。

这么看来，Chain本身其实只是个规范，就是所有可执行类，如果统一复写 `_call()` 函数，解析inputs，提供outputs，就可以在更大的框架里可以发挥价值。比如所有Chain类可以真的chain起来，因为输入输出都是`dict`，所以可以这样：
```ptyhon
# inputs 一路小跑，通过 a b c d e f 这条链，最终变成 outputs
outputs = chain_f(chain_e(chain_d(chain_c(chain_b(chain_a(inputs))))))
```


## Agent
简单扯两句Agent，详细的后面再聊。Langchain 中另外一套体系是 Agent。Chain体系，用户自己决定哪个Chain连哪个Chain，用户决定做事情的步骤，拿“检索并总结资料”这个任务来说，Chain的步骤是这样的：
* `用户提问 -> 搜索资料库获取相关资料 -> 使用LLM总结资料并给出引用 -> 展示给用户`。

但是Agent体系，用户只是抛出一个问题，LLM自己来决定要做什么：
* `用户提问 -> LLM决定怎么做`

“LLM决定怎么做” 这一步很多时候不可控，LLM也许会去检索资料并总结，也许会自己胡扯一个答案，也许会拒绝回答。这种多样性和不稳定性，也是现在LLM的一大亮点（que dian），是这种随机性让LLM有了人性的光辉（手动狗头）。


## Memory
记忆功能，通常是给LLM（大语言模型）用的。因为LLM通常有字数限制，比如ChatGPT的输入输出总共4千多个token，也就是3000多字，聊天聊得久了，就要重启，它自然就不知道之前你和它聊过什么了。这个时候，就需要一个记忆功能，把之前的聊天内容保存下来，下次聊天的时候，再把之前的内容拿出来，这样就可以聊得更长久了（ 当然具体过程不是这样简单直白的，不能直接拿出来，因为有字数限制，可能历史记录本身都超过字数限制了，所以需要做一些额外工作，比如只保留最近n条，还比如，对历史记录做个总结等等。当然这个工作也可以通过ChatGPT来完成，这个就是另外一个话题了，以后如果能聊到 SummaryMemory 的时候，再说）。

我个人有一点不太理解的是，Chain 为什么要有一个 memory 变量固定在里面，似乎有些限制 Chain 本身的作用发挥，因为Memory主要是给LLM用的。难道不是说，和 Model（语言模型）类绑定会更好一些吗？或者Memory本身也是一个Chain，这样正好Chain起来，也可以实现相同的功能。我暂时还没有搞懂作者的意图，先放一放。

类名叫 `BaseMemory` 或 `Memory`，它有两个主要的续函数，对应Chain的两个步骤：
* 首先回顾一下，核心函数 `_call()` 是这样用的：`outputs = _call(intputs)`，然后：
* `memory.save_context(inputs, outputs)` 在 Chain 的第3.步中被调用，作用是将`_call()`的 inputs 和 outputs 都保存起来。
* `memory.load_memory_variables()` 在 Chain 的第1.步中被调用，Chain 通过这个函数得到之前保存下来的值，并到inputs中，制造一个新的inputs。

Memory有一个最简单的可用的实现类，叫 `ConversationStringBufferMemory`。虽然名字长，但机理很简单：
* `save_context(inputs, outputs)`：把 inputs 中的值搞成 "Human: {input}\n"，把 outputs 的值搞成 "AI: {output}\n"，添加到一个字符串后面，于是几次回合后，这个字符串就变成了这个样子：
	```
	Human: 第一次输入的内容
	AI: 第一次回答内容
	Human: 第二次输入的内容
	AI: 第二次回答内容
	Human: 第三次输入的内容
	AI: 第三次回答内容
	```
	因为 inputs 和 outputs 都是 `dict`，所以这个类有一个前提条件，就是 inputs 和outputs，只能有一个 key，多个 key，它就不知道到底取哪个值，就直接 raise exception了。
* `load_memory_variables()`：把这个多次拼接好的字符串，返回出来。别忘了，返回值是一个 `dict`。格式是这样：`{"history"："..."}`


## 例子
先来看一个Chain和Memory的例子：

对了，先要说一下关于Chain的1. 和 3.两步。之前有提到，这两步里面会检查输入输出是否合法，怎么检查的呢，是通过 Chain 类的两个属性 `input_keys` 和 `output_keys`。
当调用 `_call()` 的时候，输入的 keys 数量，比 input_keys 只多不少，就是保证 `input_keys` 都要有。当`_call()` 返回的时候的时候，要保证 keys 数量，比`output_keys`只多不少，就是`output_keys`里的key，都要有。

因为 Chain 是不完整的，所以我们至少需要实现 `_call`、`input_keys` 和 `output_keys`才能用。`input_keys` 和 `output_keys` 是python的属性，并不是函数，具体用法可以参考python的属性用法，这里不做介绍。

```python
from langchain.chains.base import Chain

class EmptyChain(Chain):
    @property
    def input_keys(self):
        return ['input']

    @property
    def output_keys(self):
        return ['output']

    def _call(self, inputs, run_manager = None):
        return {'output': 'Hello, you!', 'other':'more than one output', 'arg_output':'output'}

```

这个Chain，只做了一件事情：被调用的时候，返回 'Hello, you!' 和 其他一些值。

效果如下：

```python
c = EmptyChain()
intputs = {'input': 'Hello, World!'}
outputs = c(intputs)
print(outputs)
#{'input': 'Hello, World!', 'output': 'Hello, you!', 'other': 'more than one output', 'arg_output': 'output'}

intputs = {'input': 'Hello, Again!', 'other': 'more than on input', 'arg_input':'input'}
outputs = c(intputs)
print(outputs)
# {'input': 'Hello, Again!', 'other': 'more than one output', 'arg_input': 'input', 'output': 'Hello, you!', 'arg_output': 'output'}

```

再来看一下 `ConversationStringBufferMemory` 的效果。因为这个类只接受一个key的 inputs 和 outputs，所以我们的 EmptyChain 需要改一下。可能需要先 `pip install strgen` 和 `pip install StringGenerator` 一下

```python
from langchain.chains.base import Chain
from strgen import StringGenerator

class EmptyChain(Chain):
    @property
    def input_keys(self):
        return ['input']

    @property
    def output_keys(self):
        return ['output']

    def _call(self, inputs, run_manager = None):
        return {'output': StringGenerator("[\l]{8}").render()}

```
然后这样创建Memory和Chain：

```python
from langchain.memory.buffer import ConversationStringBufferMemory

c = EmptyChain(memory=ConversationStringBufferMemory())

for i in range(3):
    intputs = {'input': StringGenerator("[\d]{8}").render()}
    outputs = c(intputs)
    print(outputs)

```
最后输入是这样的：

```Json
{'input': '15256240', 'history': '', 'output': 'fvqkXXYK'}
{'input': '74587513', 'history': '\nHuman: 15256240\nAI: fvqkXXYK', 'output': 'usZTTCeA'}
{'input': '53183435', 'history': '\nHuman: 15256240\nAI: fvqkXXYK\nHuman: 74587513\nAI: usZTTCeA', 'output': 'zhiruEjj'}
```

可以看到因为有了Memory，outputs里面多了一个key，叫做 `history`，这个key的值，就是之前的聊天记录，会随着Chain的调用，不断增加。

第一次 `history` 是空的，因为第一次调用的时候，还没有聊天记录。

第二次 `history` 是第一次的聊天记录:
```text

Human: 15256240
AI: fvqkXXYK'
```

第三次 `history` 是前两次的聊天记录:
```text

Human: 15256240
AI: fvqkXXYK
Human: 74587513
AI: usZTTCeA
```
那个，这些个 inputs，outputs，以及 history，都怎么用呢？下次再说吧。

Lesson 2: [说说LLM和Prompt](lesson2.md)