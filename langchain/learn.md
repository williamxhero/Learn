## BaseMemory
> 用填充input keys，保存inputs，outputs

`load_memory_variables( inputs: Dict ) -> Dict` 

`save_context( inputs: Dict, outputs: Dict )`

---
## Callbacks
> 回调函数的类列表，或者 xxxManager  
> 而xxxManager，就是 callbacks 的 Manager。比如 `RunManager` 其实就是给 callbacks 发送 `on_text` 的类

= `[BaseCallbackHandler]` or  `Manager`

---
## Chain

* `memory`： `BaseMemory`
* `callbacks`： `Callbacks`

### 执行函数：  

`input_keys()` -> `List`： = 0 
* 执行`__call__(inputs)` 时，`inputs` 比 `input_keys` 只多不少

`output_keys()` -> `List`： = 0 
* 当`__call__()` -> `outputs` 返回时，`outputs`比 `output_keys` 只多不少

`_call()`：= 0
* `inputs: Dict` -> `Dict`
* `run_manager`由`__call__`保证创建

**`run()`**：
* `inputs: *arg | **kwargs` -> `str`
* 调用 `__call__`，只返回`output_keys[0]`对应的值

**`apply()`**：
* `input_list: List[Dict]` -> `List[Dict]`
* 用每一个input 调用`__call__` 收集所有返回值
* input 是一个 list [ Dict ]
* output 是一个 list [ Dict ]

**`__call__()`**：
* `inputs: Dict|any` -> `Dict`
* `inputs` 必须指定 key-value
* `outputs` 包含 `inputs` 和 `outputs`，如果 设定 `return_only_outputs` 则只返回 `outputs`
* `memory` 和 `callback` 在这里被调用
* `prep_inputs(inputs)` -> `inputs` 
  * 如果是一个`string`参数，则做成`list`
  * 检查参数：所有参数都必须赋值 （符合 `input_keys` 数量）
  * 有 `memory`，合并 `memory` 的 `input_keys`到`inputs` `self.memory.load_memory_variables(inputs)`
* `_call(inputs，run_manager)` -> `outputs`
  * 用 `callbacks` 创建 `run_manager`
* `prep_outputs(inputs, outputs)` -> `outputs`
  * 检查返回值：所有返回值都必须赋值 （符合 `output_keys` 数量）
  * 有 `memory`，调用 `memory.save_context(inputs, outputs)`
  * 融合 `inputs` 和 `outputs`，如果 设定了`return_only_outputs` 则只返回 `outputs`

---
## LLMCHain (Chain)
> 一个LLMCHain，对应一个prompt，只有内部memory起作用。所以谁有prompt，谁才需要有memory
* `prompt` 提示语
* `llm` 语言模型
* `input_keys` 直接返回 `prompt.input_variables`
* `output_key` "text"
* `output_keys`： `[output_key]`

### 执行函数：
**`predict()`**： 
* 功能和 `run` 没有区别，只是需要 key:value
* 只返回值 `outputs[output_key]`

**`apply()`**：
* 功能复刻 `__call__` -> `_call`，但不支持 `memory`

`_call()`：
* 调用 `generate`，只传入一个 input，只接收一个 output

`generate()`：
* 接收`list`，返回`list`
* 调用 `prep_prompts`，用`inputs`填充`prompts`
* 调用 `llm.generate_prompt` 生成内容


---
## BaseLanguageModel
`generate_prompt()` ：


---
## AgentExecutor (Chain)

> memory 属于 AgentExecutor  
prompt 属于 Agent

* `agent`

### 执行函数：

`_call()`：<- `Chain.run` 触发
* `_take_next_step()`： 
  * `agent.plan()` -> `output`
  * `output` 是 `AgentFinish` 返回
  * `output` 是 `AgentAction`
    * `agent.tool.run()`


---
## Agent

* `llm_chain`： `LLMChain`
* `output_parser`： `AgentOutputParser`

`plan()`：
  * `get_full_inputs()`： 
  * `llm_chain.predict()`： 
  * `output_parser.parse()`： 



---
## StructuredChatAgent

> stop : "Observation:"

### 执行函数：
`_construct_scratchpad()`： 


---
## BaseTool

### 执行函数：

`run()`：
* `self.api_wrapper.run()`： 