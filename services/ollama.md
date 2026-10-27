# ollama

LLM inference with 4x 32gb Tesla V100s. One model per container, with LiteLLM and OpenWebUI in front that show a single API and web UI.

## Why ollama at all
The V100s are Volta, which newer inference platforms have been dropping support for while llama.cpp (what ollama runs on) still supports them. The cards also have no NVLink, so tensor parallelism across GPUs isn't on the menu. What I do have is llama.cpp splitting the model's layers across the four cards, which runs fine over plain PCIe, although slower than NVLink would be. Given this hardware, my realistic options were ollama or llama.cpp directly, and ollama won because packaging, model management and the API were already done.

## Why one model per instance
I ran two agent machines against one ollama instance packed with models. I changed the model config on one of them, and the other kept sending requests that expected the old one, so the agents effectively fought over the instance and both got unusable performance due to the constant model re-loads. One model per container prevents this; switching a model is just booting a different container, and a crash takes down one model instead of the whole inference setup.

## Why LXCs
Virtually no overhead compared to running the service on the host, and I can spin a container up or shut it down in seconds when I want a model's VRAM back. The NVIDIA userspace driver installs inside the container with `--no-kernel-modules` and the kernel module stays on the host. A VM per GPU would be better isolation, but it dedicates the whole card and boots slower than I care to wait.

## Settings
The environment I run ollama with:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434" # firewall decides who connects, not ollama
Environment="OLLAMA_KEEP_ALIVE=-1" # re-loading a 100GB model per request means minutes of latency until generation starts
Environment="OLLAMA_MAX_LOADED_MODELS=1" # ensures VRAM reserved for context isn't stolen by a second model
Environment="OLLAMA_NUM_PARALLEL=2" # two agents may share one model at once, can be raised if VRAM budget allows
```

## Naming
Hostnames are as follows: `ollama{gpus}{type}`, IP: `10.1.30.{gpus}{type}`, where type 1 = MoE, 2 = dense. For example, a dense model across three V100s is `ollama32` at `10.1.30.32`. The name + IP say which model and how many GPUs it runs on, so I never need to search through my documentation.

## Setup
GPU passthrough LXC on the compute VLAN. The NVIDIA driver installs inside the container with `--no-kernel-modules`, since the kernel module has to live on the Proxmox host.

Only the services VLAN reaches ollama. The compute VLAN can't touch the LAN at all.

## What runs on it
The 112GB model below is split across the 4x32GB of VRAM:

```
~# ollama list
NAME                  ID              SIZE      MODIFIED
qwen3.8:Flash-Next    858dddf501ff    112 GB    2 days ago
```

*nvidia-smi, all 4 cards serving one model:*
```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09             Driver Version: 580.126.09     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Tesla PG500-216                On  |   00000000:05:00.0 Off |                    0 |
| N/A   42C    P0             65W /  250W |   29018MiB /  32768MiB |     20%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   1  Tesla PG500-216                On  |   00000000:06:00.0 Off |                    0 |
| N/A   37C    P0             61W /  250W |   25658MiB /  32768MiB |     20%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   2  Tesla PG500-216                On  |   00000000:09:00.0 Off |                    0 |
| N/A   38C    P0             61W /  250W |   25908MiB /  32768MiB |     20%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   3  Tesla PG500-216                On  |   00000000:0B:00.0 Off |                    0 |
| N/A   41C    P0             63W /  250W |   25218MiB /  32768MiB |     18%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A           19855      C   ...local/lib/ollama/llama-server      29012MiB |
|    1   N/A  N/A           19855      C   ...local/lib/ollama/llama-server      25652MiB |
|    2   N/A  N/A           19855      C   ...local/lib/ollama/llama-server      25902MiB |
|    3   N/A  N/A           19855      C   ...local/lib/ollama/llama-server      25212MiB |
+-----------------------------------------------------------------------------------------+
```