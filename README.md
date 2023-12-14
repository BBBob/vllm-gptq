这是vLLM（对应版本0.22）的一个分支。在这个分支里，我们提供了gptq量化的支持，通过这个分支你可以直接加载由AutoGPTQ得到的量化模型。

## 新增功能
该版本vLLM跟官方0.22版本的vLLM主要区别在于增加gptq a16w4量化模型支持。我们在Qwen-72B上测试了量化模型性能，结果如下表。

| context length | generate length | tokens/s    | tokens/s   | tokens/s    | tokens/s   | tokens/s    | tokens/s   |   tokens/s  |  tokens/s  |
|----------------|-----------------|-------------|------------|-------------|------------|-------------|------------|:-----------:|:----------:|
|                |                 |     tp=8    |    tp=8    |     tp=4    |    tp=4    |     tp=2    |    tp=2    |     tp=1    |    tp=1    |
|                |                 | fp16 a16w16 | int4 a16w4 | fp16 a16w16 | int4 a16w4 | fp16 a16w16 | int4 a16w4 | fp16 a16w16 | int4 a16w4 |
|        1       |        2k       |    26.42    |    27.68   |    24.98    |    27.19   |    17.39    |    20.76   |      -      |    14.63   |
|       6k       |        2k       |    24.93    |    25.98   |    22.76    |    24.56   |      -      |    18.07   |      -      |      -     |
|       14k      |        2k       |    22.67    |    22.87   |    19.38    |    19.28   |      -      |    14.51   |      -      |      -     |
|       30k      |        2k       |    19.95    |    19.87   |    17.05    |    16.93   |      -      |      -     |      -      |      -     |


## 如何开始

### 安装
目前，我们仅支持源码安装。
```
   cd vllm 
   pip install -e .
```

### 如何使用
我们在此仅介绍如何运行gptq a16w4量化模型，如果想使用vllm其他功能，请阅读 [官方文档](https://github.com/vllm-project/vllm)。关于QWen量化模型的示例代码，代码目录在tests/qwen/。

#### 批处理调用模型
注意：运行以下代码，需要先进入对应的目录：tests/qwen/。

```python
from vllm_wrapper import vLLMWrapper

if __name__ == '__main__':
    model = ""
    #gptq a16w4 model
    vllm_model = vLLMWrapper(model,
                             quantization = 'gptq',
                             dtype="float16",
                             tensor_parallel_size=2)

    response, history = vllm_model.chat(query="你好",
                                        history=None)
    print(response)
    response, history = vllm_model.chat(query="给我讲一个年轻人奋斗创业最终取得成功的故事。",
                                        history=history)
    print(response)
    response, history = vllm_model.chat(query="给这个故事起一个标题",
                                        history=history)
    print(response)

```

#### API方式调用模型

注意：除去安装vllm根目录下的requirement.txt里提到的软件，以API方式调用模型需要额外安装fast chat

```bash
pip install fschat
pip install accelerate

```

##### 启动Server

step 1. 启动控制器

```
    cd FastChat/
    python -m fastchat.serve.controller
```

step 2. 启动模型worker
```
    python -m fastchat.serve.vllm_worker --model-path $model_path --tensor-parallel-size 4 --trust-remote-code
```

step 3. 启动服务器
```
   python -m fastchat.serve.openai_api_server --host localhost --port 8000
```

##### API调用

step 1. 安装openai-python
```bash
pip install --upgrade openai
```
step 2. 调用接口
```python
import openai
# to get proper authentication, make sure to use a valid key that's listed in
# the --api-keys flag. if no flag value is provided, the `api_key` will be ignored.
openai.api_key = "EMPTY"
openai.api_base = "http://localhost:8000/v1"

model = "qwen"
call_args = {
    'temperature': 1.0,
    'top_p': 1.0,
    'top_k': -1,
    'max_tokens': 2048, # output-len
    'presence_penalty': 1.0,
    'frequency_penalty': 0.0,
}
# create a chat completion
completion = openai.ChatCompletion.create(
  model=model,
  messages=[{"role": "user", "content": "Hello! What is your name?"}],
  **call_args
)
# print the completion
print(completion.choices[0].message.content)
```