# FlexGen

FlexGen is a high-throughput generation engine for running large language models with limited GPU memory.

Large language models (LLMs) are at the heart of applications like ChatGPT and Copilot, but the high computational and memory requirements of LLM inference traditionally make it feasible only with multiple high-end accelerators. FlexGen aims to lower the resource requirement of LLM inference down to a single commodity GPU and allow flexible deployment for various hardware setups.

The key features of FlexGen include:  

⚡ **Lightining Fast Offloading**.  
Up to 100x faster than other offloading-based systems for running 175B models on a single GPU.  

📦 **Extreme Compression**.  
Compress both the parameters and attention cache of models, such as OPT-175B, down to 4 bits with negligible accuracy loss.

🚀 **Scalability**.  
Come with a distributed pipeline parallelism runtime to allow scaling if more GPUs are given.

| [**Read Paper**](docs/paper.pdf) | [**Join Discord**](https://discord.gg/JfphDTkBAh) |

## Content
- [Benchmark Results](#benchmark-results)
- [Install](#install)
- [Get Started with a Single GPU](#get-started-with-a-single-gpu)
- [Run ChatOPT on a Single GPU](#run-chatopt-on-a-single-gpu)
- [Scaling to Distributed GPUs](#scaling-to-distributed-gpus)
- [Roadmap](#roadmap)

## Benchmark Results
### Generation Throughput (token/s)
| System | OPT-6.7B | OPT-30B | OPT-175B |
| ------ | -------- | ------- | -------- |
| Huggingface Accelerate   | 25.12 | 0.62 | 0.01 |
| DeepSpeed ZeRO-Inference | 9.28  | 0.60 | 0.01 |
| Petals\*                 | -     | -    | 0.05 |
| FlexGen                  | 25.26 | 7.32 | 0.69 |
| FlexGen with Compression | **29.12** | **8.38** | **1.12** |

- Hardware: an NVIIDA T4 (16GB) instance on GCP with 208GB of DRAM and 1.5TB of SSD.  
- Workload: input sequence length = 512, output sequence length = 32. The batch size is tuned to a value that maximizes the generation throughput for each system.  
- Metric: generation throughput (token/s) = number of the generated tokens / (time for processing prompts + time for generation).  

How to [reproduce](benchmark/flexgen).

### Latency-throughput Trade-off
The figure below shows the latency and throughput trade-off of three offloading-based systems on OPT-175B (left) and OPT-30B (right).
FlexGen achieves a new Pareto-optimal frontier with a 100x higher maximum throughput for OPT-175B.
Other systems cannot further increase throughput due to out-of-memory. ``(c)'' denotes FlexGen with compression.

<img src="https://github.com/Ying1123/FlexGen/blob/main/docs/throughput_vs_latency.jpg" alt="logo" width="500"></img>


## How It Works
FlexGen can be flexibly configured under various hardware resource constraints by aggregating memory and computation from the GPU, CPU, and disk. Through a linear programming optimizer, it searches for the best pattern to store and access the tensors, including weights, activations, and attention key/value (KV) cache. FlexGen further compresses both weights and KV cache to 4 bits with negligible accuracy loss. 

One key idea of FlexGen is to play the latency-throughput trade-off. Achieving low latency is inherently challenging for offloading methods, 
but the efficiency of offloading can be greatly boosted for throughput-oriented scenarios (see the figure above).
FlexGen utilizes a block schedule to reuse weight and overlap I/O with computation, as shown in figure (b) below, while other baseline systems use an ineffiicent row-by-row schedule, as shown in figure (a) below.

<img src="https://github.com/Ying1123/FlexGen/raw/main/docs/block_schedule.jpg" alt="logo" width="500"></img>

More details can be found in [our paper](docs/paper.pdf).


## Install
Requirements:
```
torch>=1.12
```

Instructions:
```
git clone https://github.com/Ying1123/FlexGen.git
cd FlexGen
pip3 install -e .

# (Optional) Install openmpi for multi-gpu execution
# sudo apt install openmpi-bin
```

## Get Started with a Single GPU

### OPT-1.3B
To get started, you can try a small model like OPT-1.3B first. It fits into a single GPU so no offloading is required.
FlexGen will automatically download weights from huggingface.
```
python3 -m flexgen.flex_opt --model facebook/opt-1.3b
```

### OPT-30B
To run large models like OPT-30B, you will need to use CPU offloading. You can try commands below. The `--percent` arguments specify the offloading strategy for parameters, attention cache and hidden states separately.
```
python3 -m flexgen.flex_opt --model facebook/opt-30b --percent 0 100 100 0 100 0
```

### OPT-175B
To run OPT-175B, you need to download the weights from [metaseq](https://github.com/facebookresearch/metaseq/tree/main/projects/OPT) and convert the weights into Alpa [format](https://alpa.ai/tutorials/opt_serving.html#convert-opt-175b-weights-into-alpa-formats).
You can then try CPU/disk offloading by
```
python3 -m flexgen.flex_opt --model facebook/opt-175b --percent 0 0 0 0 0 0 --offload-dir YOUR_SSD_FOLDER
```

### How to set the offloading strategy?
We will release an automatic policy optimizer later, but now you have to manually try a few strategies.
The idea of high-throughput generation is to offload parameters and attention cache as much as possible to CPU and disk if necessary.
You can see the reference startegies in our benchmark [here](https://github.com/Ying1123/FlexGen/blob/956859634efb9133f39b4e3fac7bfb0ce5dfadbf/benchmark/flexgen/bench_suite.py#L39-L79).

## Scaling to Distributed GPUs
If you have more GPUs, FlexGen can combine offloading with pipeline parallelism to allow scaling.
For example, if you have 2 GPUs but the aggregated GPU memory is less than the model size, you still need offloading. FlexGen allow you to do pipeline parallelism with these 2 GPUs to accelerate the generation.
See examples [here](https://github.com/Ying1123/FlexGen/tree/main/benchmark/flexgen#distributed-gpus).

## Run ChatOPT on a Single GPU
[chatbot.py](apps/chatbot.py) shows how to build a chatbot with FlexGen and OPT models.
While FlexGen is mainly optimized for throughput-oriented scenarios like dataset evaluations and information extraction, FlexGen can also be used for interactive applications like chatbot with better performance than other offloading-based systems. Note that FlexGen cannot achieve its best throughput in this single-batch case.

### Commands
```
# Chat with OPT-6.7B
python3 chatbot.py --model facebook/opt-6.7b

# Chat with OPT-30B
python3 chatbot.py --model facebook/opt-30b --percent 0 100 100 0 100 0
```

### Example output
```
A chat between a curious human and a knowledgeable artificial intelligence assistant.
Human: Hello! What can you do?
Assistant: As an AI assistant, I can answer questions and chat with you.
Human: What is the name of the tallest mountain in the world?
Assistant: Everest.
Human: I am planning a trip for our anniversary. What things can we do?
Assistant: Well, there are a number of things you can do for your anniversary. First, you can play cards. Second, you can go for a hike. Third, you can go to a museum.
```

## Roadmap
We plan to work on the following features. Community conributions are welcome.

- [ ] Support Apple silicon M1/M2 deployment
- [ ] Support Colab deployement
- [ ] Optimize the latency of the chatbot application
- [ ] Add a text summarization application
- [ ] Support more models (BLOOM, CodeGen, OPT-IML)
- [ ] Release the cost model and policy optimizer
- [ ] Release a pip installable package
