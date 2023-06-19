# Install GPT LLaMa Model on Laptop

## Objective

To run a simple GPT model on a Linux laptop and become familiar with the GPT model's capabilities.

> Reference > https://github.com/nomic-ai/gpt4all and the original GPT4All Model (based on GPL Licensed LLaMa)

## Hardware Pre-requisite

I tried this with Fedora 36 on MacBook Pro 16,2 machine __without GPU__

```sh
(gpt) [jeff@mbp ~]$ uname -a
Linux mbp 5.18.6-200.mbp.fc34.x86_64 #1 SMP PREEMPT_DYNAMIC Mon Jun 27 16:08:52 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
```

The CPU spec and quantity of processors
```sh
(gpt) [jeff@mbp ~]$ cat /proc/cpuinfo | grep "model name"
model name	: Intel(R) Core(TM) i5-1038NG7 CPU @ 2.00GHz
(gpt) [jeff@mbp ~]$ grep -c ^processor /proc/cpuinfo
8
```

The memory
```sh
(gpt) [jeff@mbp ~]$ cat /proc/meminfo | grep MemTotal
MemTotal:       16178188 kB
```

## Software Pre-requisite

I use `miniconda3` as my Python environment

```sh
(gpt) [jeff@mbp ~]$ conda --version
conda 23.1.0

(gpt) [jeff@mbp ~]$ which python; python --version
~/miniconda3/envs/gpt/bin/python
Python 3.10.10
```

## Setup 

- Download `gpt4all-lora-quantized.bin` model from https://the-eye.eu/public/AI/models/nomic-ai/gpt4all/gpt4all-lora-quantized.bin

    Or you can find unfiltered model at https://the-eye.eu/public/AI/models/nomic-ai/gpt4all/gpt4all-lora-unfiltered-quantized.bin

- Clone the code and install dependencies
    ```sh
    git clone https://github.com/nomic-ai/gpt4all.git
    cd gpt4all
    pip install -r requirements.txt
    ```
    The output looks like
    ```sh
    Successfully built deepspeed nomic pathtools
    Installing collected packages: py-cpuinfo, pathtools, ninja, hjson, 
    xxhash, wonderwords, smmap, setproctitle, sentry-sdk, pyarrow, psutil, 
    protobuf, nodelist-inflator, loguru, jsonlines, docker-pycreds, 
    dill, responses, multiprocess, gitdb, transformers, GitPython, cohere, 
    wandb, nomic, datasets, evaluate, accelerate, torchmetrics, peft, deepspeed
    ```

- Move the model bin file into `chat` directory
    ```sh
    mv gpt4all-lora-quantized.bin ~/chat/
    ```

## Execute 

```sh
(gpt) [jeff@mbp chat]$ ./gpt4all-lora-quantized-linux-x86 
main: seed = 1682149819
llama_model_load: loading model from 'gpt4all-lora-quantized.bin' - please wait ...
llama_model_load: ggml ctx size = 6065.35 MB
llama_model_load: memory_size =  2048.00 MB, n_mem = 65536
llama_model_load: loading model part 1/1 from 'gpt4all-lora-quantized.bin'
llama_model_load: .................................... done
llama_model_load: model size =  4017.27 MB / num tensors = 291

system_info: n_threads = 4 / 8 | AVX = 1 | AVX2 = 1 | AVX512 = 0 | FMA = 1 | NEON = 0 | ARM_FMA = 0 | F16C = 1 | FP16_VA = 0 | WASM_SIMD = 0 | BLAS = 0 | SSE3 = 1 | VSX = 0 | 
main: interactive mode on.
sampling parameters: temp = 0.100000, top_k = 40, top_p = 0.950000, repeat_last_n = 64, repeat_penalty = 1.300000


== Running in chat mode. ==
 - Press Ctrl+C to interject at any time.
 - Press Return to return control to LLaMA.
 - If you want to submit another line, end your input in '\'.

> Please explain me the meaning of life
Life has no single, fixed definition; it means different things to each 
person who experiences it. For some people, life is about achieving ersonal 
goals and fulfilling dreams while for others, it's all about family or 
community involvement. Regardless of what you believe the meaning of your 
own life should be, one thing remains constant: You are responsible for 
making decisions that will affect how you experience this precious gift 
we call "life."
> 

```