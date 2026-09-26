---
title: "Fast & Efficient LLM Inference with vLLM-II"
date: 2026-09-23 02:45:00
tags: courses
---

上一篇课堂笔记主要是在做模型的优化，即 Model Optimizations: To reduce model size & cost , 对应之前讲的量化和稀疏化。

让模型变小只是一半的工作 ，还有一部分就是 Inference Optimizations: To maximize throughput & efficiency

课程里主要提到了三板斧：**Batching** → GPUBusy、**PagedAttention** → ManageKVCache、**prefix caching**  → skip KV recomputation  

## 1. vllm

仍然是上一节的 GPU 架构

```
┌───────────────────────┐                                                                       
│ GPU SRAM              │ Right next to tensor cores:                            NVIDIA A100:   
│ smallest bu fastest   │ specialized hardware for matrix multiplication         20 MB, 19 TB/s 
└───────────────────────┘                                                                       
                                                                                                
┌─────────────────────────────────────┐                                          NVIDIA A100:   
│ GPU HBM                             │                                          40 GB, 1.5 TB/s
│ smaller than host memory            │                                                         
│ but much faster                     │                                                         
└───────────────────────▲─────────────┘                                                         
                        │ 12GB/s                                                                
┌───────────────────────┴───────────────────────────────┐                                       
│ Host Machine's Main Memory                            │                                       
│ CPU DRAM                                              │                                       
│ large but slow                                        │                                       
└───────────────────────────────────────────────────────┘                    
```

预测每一个 token，都要使用整个 Transformer，也就是各层的模型权重。模型权重很大，主要放在 HBM 中，而 GPU 的高速 SRAM 很小，所以大量时间花在把权重从 HBM 搬到计算单元，而 Tensor Core 反而会因为计算量不大而闲着。

我们知道训练的时候是批处理的，所以 GPU 打的很满。推理的思路也是类似，即通过 Batching 可以让同一份权重同时服务多个 token/请求，从而摊薄权重读取成本并提高 GPU 利用率。

最朴素的想法是 Static Batching，比如并行计算 5 个请求，完成后换下一批。但是很快会遇到一个问题，以为不同 request 结束长度不同，有的 5 个 tokenEND，有的 2000 个 token END. vllm 里优化为了 **Continuous Batching** ，即不断补充新的 request 进来，GPU 就会一直 BUSY 了。*课程里有比较直观的对比图*

**PagedAttention** 用来管理 KVCache ，借鉴了 os pagecache 分页的思想。最初设计是预留一块固定大小，但是因为 token 序列长度差别很大，容易出现内存碎片。现在设计为了两级的结构，将内存分为大小相同的 block，然后第一级通过 PBN（Physical Block Number，物理块号）FilledSlots 来记录和管理空余的 Block 的槽位，第二级是实际内存。每个序列维护一张"块表"（逻辑位置 → 物理块的映射，类似 page table），块可放在内存任意位置，按需回收。

如果是`END x END`之间的内存，同时没有 prefix cache 使用，则可以删除。删除用了 LRU 的思想 [Automatic Prefix Caching - vLLM](https://docs.vllm.ai/en/latest/design/prefix_caching)  

**Prefix Caching**则源自观察到 token 序列相同前缀的场景：
a. 相同用户多轮对话，比如 msgs 往往是追加的形式，因此前面的 msgs 不会变  
b. 多个用户使用相同 system prompt      

Prefix Caching 实际是对于 KVCache 思想的延伸，vLLM 默认开启。不过如果跨多个服务实例共享缓存（比如跨 Session、跨 QA 对话落到了其他推理服务实例），就需要引入分布式的架构设计解决，那就不得不面临分布式导致的性能和成本问题了。[ds](https://api-docs.deepseek.com/zh-cn/guides/kv_cache/)、[glm](https://docs.bigmodel.cn/cn/guide/capabilities/cache#%E6%96%87%E6%A1%A3%E5%86%85%E5%AE%B9%E5%A4%8D%E7%94%A8)都有对应的缓存介绍，其中[火山引擎](https://docs.volcengine.com/docs/ark/context-cache)设计的最为复杂，有隐式、显式两种缓存，且支持的模型不同，同时又引入了 Prompt Cache Key 的路由策略。这些如果不使用 sdk/harness 专门调优，尤其是显式和 key 的设计，对于普通调用的用户成本还是比较高的。

vllm 启动推理服务非常简洁：

```bash
vllm serve Qwen/Qwen3-0.6B --dtype=bfloat16 --max-model-len 4096
```
- **`vllm serve`**: 启动 vLLM 内置推理 server。加载权重到引擎（默认开启 PagedAttention、continuous
  batching、prefix caching），并通过 HTTP 暴露在 8000 端口。
- **`Qwen/Qwen3-0.6B`**: [Hugging Face Hub](https://huggingface.co/Qwen/Qwen3-0.6B) 上的模型 ID。
  首次运行从 HF 下载权重/分词器/config 到本地缓存 (`~/.cache/huggingface/hub`)，之后复用缓存。
- **`--dtype=bfloat16`**: 以 BF16 精度加载权重。
- **`--max-model-len 4096`**: 把上下文窗口（prompt + 生成）限制在 4096 tokens。

启动后提供`/v1`接口，比如`/models /chat/completions`  

同时暴露 Prometheus 兼容的 `/metrics` 端点，用于抓取一些核心指标：
- **`num_requests_running / waiting`**: 活跃 vs 排队中的请求数  
- **`gpu_cache_usage_perc`** (或 `cpu_cache_usage_perc`): KV cache 内存压力  
- **`prompt_tokens_total / generation_tokens_total`**: 累计 token 数  
- **`prefix_cache_queries_total`**: 触发 prefix caching 检索的 query 数——prefix caching 默认开启（`vllm serve` 不加 `--enable-prefix-caching` 也会开），queries 增长只代表"查了缓存"，真正是否命中复用要看 `prefix_cache_hits_total`

代码里主要是通过构造请求(并发)，然后观察上述指标来说明 vllm 实现了上述 features，不是严格的证明，不再赘述。


## 2. Measuring What Matters: Benchmarking and Evaluation

Measuring 就是要帮助回答 COST ACCURACY PERFORMANCE 的 tradeoff 问题。主要的思路是两个：
1. **性能 (Performance)** ，回答 系统服务请求有多快，提供了 GuideLLM 工具  
2. **质量 (Quality)** ，模型在真实任务上表现如何，通过了 lm_eval 工具  

[GuideLLM](https://github.com/neuralmagic/guidellm) 是 vLLM 项目出品的推理性能基准工具。可以以可控的速率发送请求，并记录每个请求的耗时。

| 指标 | 测量什么 |
|:--|:--|
| **TTFT** | Time to first token：感知响应速度 |
| **ITL** | Inter-token latency：流式输出的平滑度 |
| **E2E latency** | 总请求耗时 |
| **Throughput** | 每秒请求数和每秒 token 数 |

结果用 JSON 保存，同时为每个指标预计算了统计量（均值、百分位、min/max）

[lm_eval](https://github.com/EleutherAI/lm-evaluation-harness) 则专注于测量**任务完成情况**，主要就是通过给定问题，然后跟基准比较答得有多好。

量化后的模型，是否达标了，能够用来部署提供服务，还是要靠 Measuring 来衡量性价比。（我理解现在已经进一步到观察在具体环境里解决问题的效果。）
  
## 3. References

- [Fast & Efficient LLM Inference with vLLM: A New Course with DeepLearning.AI](https://vllm.ai/blog/2026-06-03-deeplearning-ai-vllm-course)
