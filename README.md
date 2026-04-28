# AISBench P / D 测试方法整理

> 本文将 AISBench 的测试步骤按 **P 测试** 和 **D 测试** 两类整理，并补充 **QPS 计算方法**。  

---

## 1. 参考链接

1. [AISBench Releases](https://github.com/AISBench/benchmark/releases)

2. [AISBench Docker 镜像使用说明]  
   https://github.com/AISBench/benchmark/wiki/AISBench-Docker-Images-Guidance

3. AISBench 快速入门  
   https://ais-bench-benchmark.readthedocs.io/zh-cn/latest/get_started/quick_start.html

4. AISBench Synthetic 数据集使用指南  
   https://ais-bench-benchmark.readthedocs.io/zh-cn/latest/advanced_tutorials/synthetic_dataset.html

5. AISBench 服务化稳定状态性能测试  
   https://ais-bench-benchmark.readthedocs.io/zh-cn/latest/advanced_tutorials/stable_stage.html

[百度](http://www.baidu.com "百度首页")
---

## 2. 通用环境准备（P / D 共用）

### 2.1 获取最新镜像

AISBench 官方 Releases 页面可以查看当前镜像版本；当前示例中使用的是：

```bash
ghcr.io/aisbench/aisbench_benchmark:v3.1-20260415-master_aarch64_py_310
```

如果本机尚未拉取，可先执行：

```bash
docker pull ghcr.io/aisbench/aisbench_benchmark:v3.1-20260415-master_aarch64_py_310
```

### 2.2 启动容器

```bash
docker run --name ais_bench_container -it -d --net=host   -w /benchmark   --ipc=host   -v /data/datasets:/datasets   ghcr.io/aisbench/aisbench_benchmark:v3.1-20260415-master_aarch64_py_310   bash
```

> 说明：容器工作目录设为 `/benchmark`，数据集目录挂载到 `/datasets`，后续 P / D 测试都可复用该容器。这里的数据挂载路径请按你的宿主机实际路径替换。

### 2.3 进入容器

```bash
docker exec -it ais_bench_container /bin/bash
```

### 2.4 批量创建软链接（可选）

如果需要让 `/datasets` 中已有的数据集直接出现在 AISBench 默认数据集目录中，可以批量创建软链接：

```bash
for dir in /datasets/*; do
  name=$(basename "$dir")
  ln -s "$dir" "/benchmark/ais_bench/datasets/$name"
done
```

### 2.5 查询配置文件路径（强烈建议先执行）

AISBench 官方快速入门和稳态测试文档都建议用 `--search` 来打印模型、数据集、结果汇总任务对应的配置文件绝对路径。

**P 测试查询命令：**

```bash
ais_bench --models vllm_api_stream_chat --datasets synthetic_gen_string --search
```

**D 测试查询命令：**

```bash
ais_bench --models vllm_api_stream_chat --datasets synthetic_gen_string --summarizer stable_stage --mode perf --search
```

根据当前环境的查询结果，`vllm_api_stream_chat` 的模型配置文件路径为：

```text
/benchmark/ais_bench/benchmark/configs/models/vllm_api/vllm_api_stream_chat.py
```

而 `synthetic_gen_string` 对应的数据集配置文件通常位于 `configs/datasets/synthetic` 目录下；Synthetic 数据集官方指南也说明应修改该目录下模板文件中的 `synthetic_config` 参数。

---

## 3. 公共模型配置（P / D 共用）

编辑模型配置文件：

```bash
vi /benchmark/ais_bench/benchmark/configs/models/vllm_api/vllm_api_stream_chat.py
```

AISBench 的服务化模型配置中，通常需要根据实际推理服务修改：`model`、`host_ip`、`host_port`、`url`、`batch_size`、`max_out_len` 以及 `generation_kwargs` 等参数。

建议重点确认：

- `model`：服务端已加载模型名称。
- `host_ip` / `host_port` / `url`：推理服务地址。
- `batch_size`：请求并发。
- `max_out_len`：模型最大输出 token 数。
- `generation_kwargs.ignore_eos=True`：忽略 eos，使输出长度达到 `max_out_len`。若输出长度与预期差异较大，应检查这一项是否正确配置。

一个可参考的配置片段如下（以你的实际服务参数为准）：

```python
from ais_bench.benchmark.models import VLLMCustomAPIChatStream

models = [
    dict(
        attr="service",
        type=VLLMCustomAPIChatStream,
        abbr='vllm-api-stream-chat',
        path="",
        model="",
        request_rate=0,
        retry=2,
        api_key="",
        host_ip="localhost",
        host_port=8080,
        url="",
        max_out_len=512,
        batch_size=1,
        trust_remote_code=False,
        generation_kwargs=dict(
            temperature=0.01,
            ignore_eos=True,
        )
    )
]
```

---

## 4. P 测试方法

### 4.1 测试目标

P 测试使用 `synthetic_gen_string` 合成字符串数据集，并将 **输出长度固定为 1**，用于隔离/突出输入阶段的测试特征。

### 4.2 数据集配置文件

先执行：

```bash
ais_bench --models vllm_api_stream_chat --datasets synthetic_gen_string --search
```

然后编辑查询结果里对应的 `synthetic_gen_string` 配置文件。

### 4.3 P 测试推荐配置

P 测试时，**输出长度设置为 1**；输入长度按目标测试值设置。可以用 `uniform` 固定长度：

```python
synthetic_config = {
    "Type": "string",
    "RequestCount": 1000,
    "StringConfig": {
        "Input": {
            "Method": "uniform",
            "Params": {
                "MinValue": 256,
                "MaxValue": 256
            }
        },
        "Output": {
            "Method": "uniform",
            "Params": {
                "MinValue": 1,
                "MaxValue": 1
            }
        }
    }
}
```

> 说明：`Type="string"`、`RequestCount`、`StringConfig.Input`、`StringConfig.Output` 都是核心配置项。`uniform` 分布下，当 `MinValue=MaxValue` 时，可实现固定长度。

### 4.4 P 测试执行命令

```bash
ais_bench --models vllm_api_stream_chat --datasets synthetic_gen_string
```

如果需要在屏幕上直接看到更详细的过程日志，可以加 `--debug`：

```bash
ais_bench --models vllm_api_stream_chat --datasets synthetic_gen_string --debug
```

### 4.5 P 测试检查点

如果结果里的输出长度与预期不一致，优先检查：

- `synthetic_config` 中 `Output` 的长度配置是否真的是 `1`。
- 模型配置中的 `max_out_len` 是否过小或与测试预期冲突。
- `generation_kwargs.ignore_eos` 是否设为 `True`。

---

## 5. D 测试方法

### 5.1 测试目标

D 测试参考 AISBench 官方“服务化稳定状态性能测试”中的“稳态测试快速入门”。执行命令时通过 `--summarizer stable_stage --mode perf` 按稳态方式统计性能结果；如果数据集规模太小难以进入稳定阶段，还可以额外启用 `--pressure` 和 `--pressure-time`。

### 5.2 D 测试的长度原则

D 测试时，**输入长度和输出长度都使用实际的值**。也就是说，不再像 P 测试那样把输出长度固定为 1，而是把 `synthetic_gen_string` 里的 `Input` 和 `Output` 都配置为实际要模拟/测量的真实长度。

### 5.3 D 测试数据集配置示例

如果要测试某个真实长度组合（例如输入 2048、输出 512），可以这样配置：

```python
synthetic_config = {
    "Type": "string",
    "RequestCount": 1000,
    "StringConfig": {
        "Input": {
            "Method": "uniform",
            "Params": {
                "MinValue": 2048,
                "MaxValue": 2048
            }
        },
        "Output": {
            "Method": "uniform",
            "Params": {
                "MinValue": 512,
                "MaxValue": 512
            }
        }
    }
}
```

> 你也可以把 `2048` 和 `512` 替换成业务中实际需要测试的输入、输出长度；如果要模拟分布而非固定值，也可以改成 `gaussian` 或 `zipf`。

### 5.4 D 测试执行命令（稳态方式）

推荐命令：

```bash
ais_bench --models vllm_api_stream_chat --datasets synthetic_gen_string --summarizer stable_stage --mode perf
```

第一次执行建议带 `--debug`，便于直接看到请求推理服务过程中的日志与报错：

```bash
ais_bench --models vllm_api_stream_chat --datasets synthetic_gen_string --summarizer stable_stage --mode perf --debug
```

### 5.5 D 测试进入稳态不足时的增强方式

如果数据集规模太小、服务难以进入稳定阶段，可以启用压力测试能力：

- 增加 `--pressure`
- 使用 `--pressure-time` 指定持续时间（不超过 86400 秒）
- 通过模型配置里的 `request_rate` 控制每个进程新增线程/客户端的频率

例如：

```bash
ais_bench --models vllm_api_stream_chat   --datasets synthetic_gen_string   --summarizer stable_stage   --mode perf   --pressure   --pressure-time 30
```

### 5.6 D 测试结果查看

稳态测试最终会输出稳态阶段的性能指标，如：

- `E2EL`
- `TTFT`
- `TPOT`
- `ITL`
- `InputTokens`
- `OutputTokens`
- `Benchmark Duration`
- `Request Throughput`
- `Output Token Throughput`

其中：

- `Current exp folder` 日志会提示本次实验的输出根目录。
- `performances/` 或 `performance/` 目录下会保存 CSV、JSON、明细 JSON / H5 以及并发可视化 HTML。
- `stable_stage` 的统计逻辑关注的是服务达到最大并发后的稳定阶段。

---

## 6. P / D 测试对比总结

### P 测试

- 数据集：`synthetic_gen_string`
- 输出长度：**固定为 1**
- 输入长度：按目标值设置
- 执行方式：`ais_bench --models vllm_api_stream_chat --datasets synthetic_gen_string`

### D 测试

- 数据集：继续使用 `synthetic_gen_string`
- 输入、输出长度：**都设置为实际值**
- 结果统计方式：`--summarizer stable_stage --mode perf`
- 若难以达到稳态：可启用 `--pressure --pressure-time`

---

## 7. QPS 计算方法

PD 分离场景是在 **TTFT、TPOT 约束条件** 下，尽可能提升吞吐，并分别计算 **P**、**D** 的 QPS。

例如：

- 约束条件示例：`TTFT <= 5s`、`TPOT <= 100ms`
- 输入长度 256、输出长度 256 场景下，需要分别测算 P 和 D 的 QPS

### 7.1 P 的计算方法

以“输入长度 256、输出长度 256”场景为例：

在测试 P 时，需要将 **输出长度设置为 1**，然后进行递增并发测试，测试出最优的 QPS。

#### 步骤 1：通过 AISBench 测试出总的 Input Token Throughput（TPS）

即先执行 P 测试，获取测试结果中的：

```text
Input Token Throughput
```

#### 步骤 2：计算 P 节点单卡 TPS

计算方式：

```text
P节点单卡TPS = Input Token Throughput / P节点卡数
```

#### 步骤 3：计算 P 节点 QPS

计算方式：

```text
P节点QPS = P节点单卡TPS × P节点卡数 / Input_len
```

> 其中：
>
> - `Input Token Throughput` 为 AISBench 结果中的输入吞吐（token/s）
> - `P节点卡数` 为 P 节点使用的卡数
> - `Input_len` 为输入长度，例如 256

### 7.2 D 的计算方法

在保持与 P 测试条件一致的前提下，使用如下方法测试 D 的 QPS。

以“输入长度 256、输出长度 256”场景为例：

- 输入长度使用实际值
- 输出长度使用实际值
- 执行 D 测试（稳态 / 稳态 + 压测）
- 结果取值使用：

```text
Request Throughput
```

即：

```text
D节点QPS = Request Throughput
```

---

## 8. 最终可直接执行的命令清单

### 8.1 启动容器

```bash
docker run --name ais_bench_container -it -d --net=host   -w /benchmark   --ipc=host   -v /data/datasets:/datasets   ghcr.io/aisbench/aisbench_benchmark:v3.1-20260415-master_aarch64_py_310   bash
```

### 8.2 进入容器

```bash
docker exec -it ais_bench_container /bin/bash
```

### 8.3 批量创建软链接

```bash
for dir in /datasets/*; do
  name=$(basename "$dir")
  ln -s "$dir" "/benchmark/ais_bench/datasets/$name"
done
```

### 8.4 P 测试：查询配置路径

```bash
ais_bench --models vllm_api_stream_chat --datasets synthetic_gen_string --search
```

### 8.5 P 测试：正式执行

```bash
ais_bench --models vllm_api_stream_chat --datasets synthetic_gen_string
```

### 8.6 D 测试：查询配置路径

```bash
ais_bench --models vllm_api_stream_chat --datasets synthetic_gen_string --summarizer stable_stage --mode perf --search
```

### 8.7 D 测试：正式执行（稳态）

```bash
ais_bench --models vllm_api_stream_chat --datasets synthetic_gen_string --summarizer stable_stage --mode perf
```

### 8.8 D 测试：正式执行（稳态 + 压测）

```bash
ais_bench --models vllm_api_stream_chat --datasets synthetic_gen_string --summarizer stable_stage --mode perf --pressure --pressure-time 30
```

---

## 9. 备注

- `--search` 只用于查询配置文件路径。
- `synthetic_gen_string` 支持通过 `uniform`、`gaussian`、`zipf` 等方式控制长度分布；若只想固定长度，最简单的方式就是把 `MinValue` 和 `MaxValue` 设为同一个值。
- D 测试使用稳态统计时，核心点不只是“跑性能”，更是按 `stable_stage` 对稳定阶段内的请求进行性能汇总。
