# Local-Deployed LLM Production Guide

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=4 orderedList=false} -->

<!-- code_chunk_output -->

- [Objective](#objective)
- [Environment and Pre-requisite](#environment-and-pre-requisite)
    - [Hardware and Env](#hardware-and-env)
    - [Models Location](#models-location)
    - [Docker Image Location](#docker-image-location)
- [Docker with Nvidia](#docker-with-nvidia)
- [vLLM with Docker](#vllm-with-docker)
    - [Docker behind Proxy](#docker-behind-proxy)
- [Models](#models)
    - [Embedding Model](#embedding-model)
- [Uninstall and Clean-up the](#uninstall-and-clean-up-the)
    - [Models from HuggingFace](#models-from-huggingface)
    - [Docker Images](#docker-images)
    - [Ollama](#ollama)

<!-- /code_chunk_output -->



## Objective

- All design, deployment consideration is for **production**
- Run everything as regular service for standard operation, instead of prototype or demo

## Environment and Pre-requisite

#### Hardware and Env

item | command | spec
-- | -- | --
distro | `cat /etc/*release* \| grep "DISTRIB_DESC"` | `Ubuntu 22.04.5 LTS`
kernel | `uname -r` | `5.15.0-126-generic`
ram | `cat /proc/meminfo` | `MemTotal:       32665652 kB`
gpu | `nvidia-smi --query-gpu=gpu_name,memory.total --format=csv` | `Tesla T4, 15360 MiB`
docker | `docker -v` | `Docker version 27.3.1, build ce12230`


#### Models Location

Since I use `huggingface` as model controller, all models go to `~/.cache/huggingface/hub/`


#### Docker Image Location

`/var/lib/docker/`


## Docker with Nvidia

https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

## vLLM with Docker

https://docs.vllm.ai/en/latest/serving/deploying_with_docker.html

```sh
export HUGGING_FACE_HUB_TOKEN=${HUGGINGFACEHUB_API_TOKEN}

docker run -d --name vllm --runtime nvidia --gpus all \
    -v ~/.cache/huggingface/:/root/.cache/huggingface \
    --env "HUGGING_FACE_HUB_TOKEN=${HUGGINGFACEHUB_API_TOKEN}" \
    --env HTTP_PROXY=http://127.0.0.1:10809 \
    --env HTTPS_PROXY=http://127.0.0.1:10809 \
    -p 8000:8000 --ipc=host vllm/vllm-openai:latest \
    --model BAAI/bge-large-zh-v1.5 --task embedding
```

#### Docker behind Proxy 

Place the following into `~/.docker/config.json` if you have to access through a proxy

```json
{
 "proxies": {
   "default": {
     "httpProxy": "http://172.17.0.1:10809",
     "httpsProxy": "http://172.17.0.1:10809",
     "noProxy": "*.test.example.com,.example.org,127.0.0.0/8"
   }
 }
}
```

## Models

#### Embedding Model

```py
from langchain_huggingface import HuggingFaceEmbeddings
embedding = HuggingFaceEmbeddings(
    # model_name = "sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2"
    model_name = "BAAI/bge-large-zh-v1.5"
)
```

## Uninstall and Clean-up the 

#### Models from HuggingFace

`~/.cache/huggingface/hub`

#### Docker Images

`/var/lib/docker`

`docker image ls`

`docker image rm`

#### Ollama
