# 批量调用大模型的技巧

![PDF](../assets/批量调用LLM.pdf){ type=application/pdf style="min-height:50vh;width:100%" }

!!! info "碎碎念"
    这个 slide 是我用 Typst + touying 做的，这比 PowerPoint 更 cooool！
    所以我在想能否利用 Typst 生态把 slide 嵌到这个文档界面，我尝试了用 Typst 编译出 SVG 文件，然而发现 SVG 不能拷贝出文本，这就很蠢了，还不如直接放个 PDF 窗口在这儿。 

## 引言

假设我们有一项需要 LLM 完成的任务和一批数据，希望通过调用 API 的方式获得 LLM 输出来处理这批任务。比如，以分类任务为例，对每一个样本，需要根据提示词模版构建提示词，然后发送一个请求给 OpenAI 等 LLM 服务提供方，再解析输出得到分类结果。

这个过程看似非常简单直观，但在实践中可能会面对诸多问题，尤其是在数量较大时（比如 10 万以上样本量，全部请求总量超过 10 million tokens）...

1. **如何获得百分百可靠的输出结果？** 任务需要解析生成的文本，得到分类标签。LLM 不一定总会遵循指令，而且可能还要考虑 COT 生成、模型拒答等情况，纯文本解析显得不够可靠。
2. **如何实现并发？** 对于单个生成任务，LLM 的响应速度是很难提升的，所以几万样本顺序执行的时间开销可能会非常大。我们知道，无论是 OpenAI 等 API 服务商还是本地部署的 LLM 推理，都支持并发处理，且效率会高很多。这时我们就需要在客户端实现优雅的并发请求逻辑。
3. **如何缓存调用结果避免浪费 API 费用？** 大批量调用 API 会产生不菲的开销。我们可能会中断任务处理问题、调整实现，但我们不希望已经请求过的内容“浪费掉”，所以把每次请求的结果进行缓存，不仅能提高重复执行时的速度，还能避免浪费 token 费用。
4. **如何处理各类异常？** 脚本执行过程中可能会产生各种错误，从最常见的网络错误，到请求构建和结果解析，都有可能抛出异常。我们不希望批量执行被任何错误中断，所以需要充分考虑可能会出现的异常类型并加以处理。

针对以上问题，本文讨论一些简单有效的解决方案，并附上 Python 实现。

## 准备工作

我们使用 LangChain 来简化一些功能的实现。我们的任务只涉及单次 LLM 调用，所以手工实现相关逻辑也是不难的，用 LangChain 仅仅是起到缩减代码量的效果。当然如果有更复杂的任务逻辑，相信 LangChain 也能帮到很多。

```bash
pip install langchain langchain-openai langchain-community
```

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnableLambda
from langchain_core.messages import AIMessage
from langchain_community.cache import SQLiteCache
```

我们假设数据是 JSON Lines 格式，每一行分别代表一个样本，每个样本都由固定格式的 JSON 对象描述，非常灵活易用。同时，相比于 JSON，每个文本行一个样本也可以避免一次读入全部文件到内存，IO 占用时间。

大模型服务方面，我们采用最典型的 OpenAI API 格式，因为许多国内厂商都会与之兼容。

## 结构化输出

首先，引导大模型生成结构化输出能让我们的任务更加可靠，具体来说就是让 LLM 输出 JSON 文本，然后再解析出 JSON 对象的字段得到答案。

今年八月，OpenAI 发布了 structured outputs 特性，允许开发者指定输出格式，模型会百分比可靠地按 JSON schema 输出结果 [^1]。不过考虑到不是所有平台都支持 json schema，我们这次就只使用普通的 JSON 输出。首先，在提示词部分提供指令和示例输出：

```markdown
## Response Format

Your response should be in JSON format, containing fields: `reasoning_steps`, steps that analyze provided content and lead to a classification conclusion, and `category`, must be one of the candidate categories. The steps should be concise and accurate as much as you can. Here is an example of the desired format:

{{
  "reasoning_steps": [
    "first...",
    "then..."
  ],
  "category": <label>
}}
```

我们引导模型先生成一系列 “推理步骤”，然后再生成标签字段 “category” ，这是利用了思维链技巧提高准确性。

使用 LangChain 创建一个 LLM 抽象：

```python
llm = ChatOpenAI(
	model="...",
	api_key="...",
	base_url="...",
	temperature=0.0,
	top_p=0.001,
).bind(response_format={"type": "json_object"})
```

创建提示词模版对象。我们把提示词放在一个本地文件方便编辑，注意文本中的 `{field}` 是可被替换的模版占位符，要想使用真正的花括号需要使用两个花括号来转义。

```python
from pathlib import Path
prompt = Path("./prompt.md").read_text()
prompt_template = ChatPromptTemplate.from_messages([
	("system", "You are a helpful assistant."),
	("user", prompt),
])
```

然后我们需要编写处理输出的逻辑。LangChain 对象 `ChatOpenAI` 调用 `invoke()` 会返回一个 `AIMessage` 对象，最简单的处理方式如下：

```python
def parse_response(response: AIMessage) -> dict:
    parsed = json.loads(response.content)
    return {"category": parsed["category"]}

chain = prompt_template | llm | RunnableLambda(parse_response)
```

以上代码创建了一个 chain，它会把输入传给提示词模版，生成提示词后调用 LLM，然后再调用我们自定义的 `parse_response` 方法，解析 JSON 字符串，最终返回一个字典代表输出。

进一步地，我们还可以获取 LLM 对该类别的概率（虽然这有别于机器学习中的类别概率），这可以通过 logprob 字段实现 [^2]。由于大模型会按格式输出 `"category": "<label>"` ，所以我们需要借助 `"category": "` 前缀来找到 `<label>` 序列中的的第一个 token，实现方式如下：

```python
llm = ChatOpenAI(
    ...
    logprobs=True)

def find_target_token(token_list: list[dict]):
	pattern = [' "', 'category', '":', ' "']
	pattern_length = len(pattern)
	# Loop through the token list with a sliding window
	for i in range(len(token_list) - pattern_length):
		current_tokens = [token_dict['token'] for token_dict in token_list[i:i + pattern_length]]
		if current_tokens == pattern:
			if i + pattern_length < len(token_list):
				return token_list[i + pattern_length]
	return None

def parse_response(response: AIMessage) -> dict:
	res = find_target_token(response.response_metadata["logprobs"]["content"])
	answer_logprob = float(res["logprob"]) if res else -0.0
	answer_prob = math.exp(answer_logprob)
	...
```

这样，我们就实现了解析 JSON 结构化输出，获取类别标签 + 类别概率。

## 异常处理

首先要考虑的是输出解析部分的异常情况。当输出解析部分出现任何问题时，我们记录该问题，然后返回解析前的原始消息：

```python
def parse_response(response: AIMessage) -> dict:
	try:
		res = find_target_token(response.response_metadata["logprobs"]["content"])
		answer_logprob = float(res["logprob"]) if res else -0.0
		answer_prob = math.exp(answer_logprob)
		parsed = json.loads(response.content)
		steps = "\n".join(parsed["reasoning_steps"])
		answer = str(parsed["category"])
		return {"category:": answer, "prob": answer_prob, "steps": steps}
	except Exception as e:
		logger.warning(f"LLM response parsing error: {e}")
		return {"category:": None, "raw": response.content}
```

这样，程序不会因为模型输出部分的问题而中断，后续也可以追溯这些意外情况。

对于 API 相关的异常，例如网络问题、rate limit 问题等，实际上 LangChain 提供了相关逻辑，它默认会在网络问题发生时进行重试，重试次数可以通过 `BaseChatOpenAI.max_retries` 参数控制 [^3]。而速率限制处理行为可以通过指定 `rate_limiter` 来控制 [^4]。

在处理异常情况时，我们难免会重新运行，当数据量大时，调用 LLM 的费用就显得浪费不起了。我们可以在 API 调用处增加一层缓存，这样我们的批量任务可以随时中断调试，不用担心任何浪费。所幸的是 LangChain 已经提供了相关功能：

```python
llm_response_cache_dir = Path(".langchain.db").as_posix()
llm_cache = SQLiteCache(database_path=llm_response_cache_dir)
llm = ChatOpenAI(
    ...
    cache=llm_cache,
)
```

这段 `llm_cache` 做的事情实际上是把每一次面向 LLM API 的请求内容和结果都做键值对缓存，当发现完全相同的请求时，直接返回结果，不走网络 API，从而实现节省时间和费用的效果。缓存使用一个本地 sqlite 文件实现。

最后，当真正执行大批量任务，不希望被任何异常中断，我们可以在最上层代码加一层捕获，比如：

```python
try:
    result = chain.invoke(inputs)
    return {**result, **inputs}
except Exception as e:
    logger.exception(f"Error: {e}")
    return {"error": str(e), **inputs}
```

## 基于协程的并发请求

假设已经定义好 `chain` 对象，我们的任务执行起来是这样：

```python
tasks: list[dict]
results = list()
for task in tasks:
    result = chain.invoke(task)
    results.append(result)
```

这种串行方式会挨个执行等待服务端生成，而 LLM 的生成速度是很慢的。要解决这个问题，OpenAI 提供了 Batches API，但有些情况下我们希望快速得到结果，或者部署服务不支持 Batch 调用形式，这时我们就非常需要自行实现并发请求。

写并发，Python 协程是一个很好的东西，实现的时候还要考虑几个点：
1. 数据量可能会非常大，避免一次性把整个数据集都创建出协程
2. 虽然 LangChain 已经帮我们处理了 rate limits，但并发数需要自己做控制
3. 怎么收集协程结果做进度条显示，毕竟没进度条还是不放心

我们一步一步来，首先把 chain 逻辑写成异步函数，使用其异步方法：

```python
async def run_task(task: dict[str, str]) -> dict[str, str]:
    try:
        result = await chain.ainvoke(task)
        return {**input_dict, **result,}
    except Exception as e:
        logger.exception(f"Error: {e}")
        return {**input_dict, "error": str(e),}
```

这个函数的输入直接对应我们的 JSON 输入对象，输出也是字典格式，非常灵活。并且这个异步函数保证不会抛异常（假设我们预料到的异常处理都已经写进 `chain` 里面了）。

然后，输入部分要分块进行，为此，我们把文件读取写成一个生成器函数，避免过大的内存开销。

```python
def read_file_in_chunks(filepath: str, chunk_size: int):
    current_batch = []
    with open(filepath, 'r', encoding="utf-8") as file:
        for line in file:
            current_batch.append(line.rstrip('\n'))
            
            if len(current_batch) >= chunk_size:
                yield current_batch
                current_batch = []
        
        # Don't forget the last partial batch
        if current_batch:
            yield current_batch
```

调用这个生成器时，我们在循环内部做 JSON 解析转字典格式。

```python
for chunk in read_file_in_chunks(filepath, 1000):
    task_dicts = [json.loads(json_text) for json_text in chunk]
```

这时，对于这一组任务，我们可以创建一批协程，也就是上面定义的 `run_task`，创建之后使用 `gather()` 来并发执行协程。例如这样：

```python
results = await asyncio.gather(*[run_task(d) for d in task_dicts])
```

为了控制并发数，很显然这还不够。我们可以使用信号量机制来控制同一时刻能够并发执行的协程数目，为此我们需要在 `run_task` 外面套一层异步函数：

```python
semaphore = asyncio.Semaphore(max_concurrency)

async def do(data):
	async with semaphore:
		return await run_task(data)
```

这里用到了 `Semaphore` 对象的 context manager 语法，在进出该 with 块时会自动进行获取 / 释放操作。在 `gather()` 调用时，所有 `do()` 协程都被启动，但只有 `max_concurrency` 个能够进入 `with` 块，其他所有都会卡在 `with` 入口等待信号量释放，如此实现请求并发数的控制。

进一步地，我们还可以在此基础上实现进度显示。我们使用来自 `rich` 库的 `rich.progress.Progress` 对象，在终端实时显示进度。我们需要把进度更新放在 `run_task()` 返回结果的位置上。

```python
progress_bar = progress.Progress()
with progress_bar:
	task_id = progress_bar.add_task()
	
	async def do(data: dict) -> dict:
		async with semaphore:
			result = await run_task(data)
			progress_bar.advance(task_id)
			return result
```

我们完全不用担心 `progress_bar` 对象产生争用。

## 完整代码

很多功能都有现有库来实现了，所以最终代码量很小，只有 100 多行，可以实现我们开头提到的各种功能。

```python
import json
import math
import asyncio
from pathlib import Path

from loguru import logger
from rich import progress
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnableLambda
from langchain_core.messages import AIMessage
from langchain_community.cache import SQLiteCache


progress_bar_columns = [
    progress.SpinnerColumn(),
    progress.TextColumn("[bold blue]{task.description}"),
    progress.TextColumn("[bold blue]{task.fields[filename]}"),
    progress.BarColumn(),
    progress.MofNCompleteColumn(),
    progress.TaskProgressColumn(),
    progress.TimeElapsedColumn(),
    progress.TimeRemainingColumn(),
]


def build_chain(
        use_llm_cache: bool = True,
        request_logprobs: bool = True,
        ):
    if use_llm_cache:
        llm_response_cache_dir = Path(".langchain.db").as_posix()
        llm_cache = SQLiteCache(database_path=llm_response_cache_dir)
    else:
        llm_cache = False

    llm = ChatOpenAI(
        model="xxx",
        api_key="xxx",
        base_url="xxx",
        temperature=0.0,
        top_p=0.001,
        max_tokens=500,
        logprobs=request_logprobs,
        cache=llm_cache,
        streaming=False,
    ).bind(response_format={"type": "json_object"})

    prompt = Path("./prompt.md").read_text()
    prompt_template = ChatPromptTemplate.from_messages([
        ("system", "You are a helpful assistant."),
        ("user", prompt),
    ])

    def find_target_token(token_list: list[dict]):
        pattern = [' "', 'category', '":', ' "']
        pattern_length = len(pattern)
        # Loop through the token list with a sliding window
        for i in range(len(token_list) - pattern_length):
            current_tokens = [token_dict['token'] for token_dict in token_list[i:i + pattern_length]]
            if current_tokens == pattern:
                if i + pattern_length < len(token_list):
                    return token_list[i + pattern_length]
        return None

    def parse_response(response: AIMessage) -> dict:
        try:
            res = find_target_token(response.response_metadata["logprobs"]["content"])
            answer_logprob = float(res["logprob"]) if res else -0.0
            answer_prob = math.exp(answer_logprob)
            parsed = json.loads(response.content)
            steps = "\n".join(parsed["reasoning_steps"])
            answer = str(parsed["category"])
            return {"category:": answer, "prob": answer_prob, "steps": steps}
        except Exception as e:
            logger.warning(f"LLM response parsing error: {e}")
            return {"category:": None, "raw": response.content}
    
    return prompt_template | llm | RunnableLambda(parse_response)


async def run_task(chain, task: dict[str, str]) -> dict[str, str]:
    try:
        result = await chain.ainvoke(task)
        return {**task, **result,}
    except Exception as e:
        logger.exception(f"Error: {e}")
        return {**task, "error": str(e),}


def read_file_in_chunks(filepath: str, chunk_size: int):
    current_batch = []
    with open(filepath, 'r', encoding="utf-8") as file:
        for line in file:
            current_batch.append(line.rstrip('\n'))
            
            if len(current_batch) >= chunk_size:
                yield current_batch
                current_batch = []
        
        # Don't forget the last partial batch
        if current_batch:
            yield current_batch


async def process_tasks_parallel(filepath: str, max_concurrency: int, chunk_size: int):
    chain = build_chain()
    total_lines = sum(1 for _ in open(filepath, 'r'))
    semaphore = asyncio.Semaphore(max_concurrency)
    progress_bar = progress.Progress(*progress_bar_columns)
    with progress_bar:
        task_id = progress_bar.add_task(
            description="Processing",
            total=total_lines,
            filename=Path(filepath).name,
        )
        
        async def do(chain, data: dict) -> dict:
            async with semaphore:
                result = await run_task(data)
                progress_bar.advance(task_id)
                return result
        
        # --- 主循环 ---
        for chunk in read_file_in_chunks(filepath, chunk_size):
            task_dicts = [json.loads(json_text) for json_text in chunk]
            results = await asyncio.gather(*[do(chain, d) for d in task_dicts])
            # 保存结果
            ...


async def main():
    await process_tasks_parallel("./dataset.jsonl", 64, 1000)


if __name__ == '__main__':
    asyncio.run(main())

```

## 参考资料

[^1]: https://openai.com/index/introducing-structured-outputs-in-the-api/
[^2]: https://python.langchain.com/docs/how_to/logprobs/
[^3]: https://python.langchain.com/docs/concepts/chat_models/#standard-parameters
[^4]: https://python.langchain.com/docs/how_to/chat_model_rate_limiting/
