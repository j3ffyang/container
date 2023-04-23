# Install ChatGLM-6B on Single GPU-Linux

## Objective

To develop a language model for English and Chinese based on ChatGLM with 6B parameters (1 trillion tokens training data), and fine-tune it for LoRA (Low Resource Applications).

Get familiar with the capabilities of the model, especially for LoRA fine-tuning.

> Reference: ChatGLM-6B is an open bilingual language model based on General Language Model (GLM) framework, with 6.2 billion parameters. With the quantization technique, users can deploy locally on consumer-grade graphics cards (only 6GB of GPU memory is required at the INT4 quantization level).

## Hardware Spec

```sh
(gpt2) jeff@gamer:~/miniconda3/envs$ nvidia-smi 
Sat Apr 22 17:36:11 2023       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 470.161.03   Driver Version: 470.161.03   CUDA Version: 11.4     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA GeForce ...  On   | 00000000:01:00.0 Off |                  N/A |
|  0%   34C    P8    15W / 220W |   6744MiB /  7979MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|    0   N/A  N/A       885      G   /usr/lib/xorg/Xorg                 86MiB |
|    0   N/A  N/A      1036      G   /usr/bin/gnome-shell               12MiB |
|    0   N/A  N/A      4165      C   python                           6641MiB |
+-----------------------------------------------------------------------------+
```

## Software Env in `miniconda` on Debian 11

```python
conda list | grep -E 'torch|transformers|nvidia'
nvidia-cublas-cu11        11.10.3.66               pypi_0    pypi
nvidia-cuda-cupti-cu11    11.7.101                 pypi_0    pypi
nvidia-cuda-nvrtc-cu11    11.7.99                  pypi_0    pypi
nvidia-cuda-runtime-cu11  11.7.99                  pypi_0    pypi
nvidia-cudnn-cu11         8.5.0.96                 pypi_0    pypi
nvidia-cufft-cu11         10.9.0.58                pypi_0    pypi
nvidia-curand-cu11        10.2.10.91               pypi_0    pypi
nvidia-cusolver-cu11      11.4.0.1                 pypi_0    pypi
nvidia-cusparse-cu11      11.7.4.91                pypi_0    pypi
nvidia-nccl-cu11          2.14.3                   pypi_0    pypi
nvidia-nvtx-cu11          11.7.91                  pypi_0    pypi
torch                     2.0.0                    pypi_0    pypi
transformers              4.27.1                   pypi_0    pypi
```

## Clone the code and download the model

```sh
git clone https://github.com/THUDM/ChatGLM-6B
cd ChatGLM-6B
pip install -r requirements.txt
```

After model is downloaded, the sharded bin files go to 

```sh
(gpt2) jeff@gamer:~/miniconda3/envs/gpt2$ pwd
/home/jeff/miniconda3/envs/gpt2
(gpt2) jeff@gamer:~/miniconda3/envs/gpt2$ du -ksh
6.3G	.
```

```sh
conda install cudatoolkit=11.3 -c nvidia
```

## Launch

- Turn on `gradio` on webUI, edit `web_demp.py` and update `share=true`

```py
demo.queue().launch(share=True, inbrowser=True)
```

## Troubleshooting

- Error message > `RuntimeError: Library cudart is not initialized`

    The recommended solution: to install `cudatoolkit`, as described earlier