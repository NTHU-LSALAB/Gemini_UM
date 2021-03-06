# Gemini_UM

## Prerequisites

- NVIDIA DRIVER
- CUDA 
- PyTorch
## About

Gemini_UM is an efficient GPU resource sharing system with fine-grained control for Linux platforms.

It shares a NVIDIA GPU among multiple clients with specified resource constraint, and works seamlessly with any CUDA-based GPU programs. Besides, it is also work-conserving and with low overhead, so nearly no compute resource waste will happen.

Furthermore, Gemini_UM intercepts the cuda memory allocation call, and let the gpu programs to use unified memory. With unified memory, we can share more jobs on the same GPU.

## System Structure

Gemini consists of three parts: *scheduler*, *pod manager* and *hook library*.

- *scheduler* (*GPU device manager*) (`gem-schd`): A daemon process managing token. Based on information provided in resource configuration file (`resource-config.txt`), scheduler determines whom to give token. Clients can launch CUDA kernels only when holding a valid token.
- *hook library* (`libgemhook.so.1`): A library intercepting CUDA-related function calls. It utilizes the mechanism of `LD_PRELOAD`, which forces our hook library being loaded before any other dynamic linked libraries.
- *pod manager* (`gem-pmgr`): A proxy for forwarding messages to applications/scheduler. It act as a client to scheduler, and every application sending requests to scheduler via this pod manager shares the token.

Currently we use *TCP socket* as the communication interface between components.

## Unified memory optimization
Although with Unified memory, we can share more jobs on the same GPU, it comes with some overhead. The overhead is mostly come from the data transfer when context switch. And our goal is to improve the aggregate throughput when sharing multiple Deep-learning jobs on a GPU. So based on the properties of Deep-learning jobs(refers to the paper: [Salus: Fine-Grained GPU Sharing Primitives for Deep Learning Applications](https://arxiv.org/abs/1902.04610)), we propose a method to reduce the overhead for the Pytorch frameworck based Deep-learning program. 
According to the Salus, we can divide DL memory allocations into :
* Model: holding model parameters, persistent region, fixed sized during job life.
* Ephemeral: temporary data during each iteration, outputs of middle layers. Only needed during computations and released between iterations.
* Framework-internal: persistent region, fixed sized, used by framework for book-keeping or data preparation pipelined.
And Persistent <<< Ephemeral memory. 

Furthermore, Pytorch use caching memory allocator to manage memory, it will cache the memory for fast memory deallocation without device synchronizations. This mechanism is useful when only one job on the GPU, but if we want to share the GPU, it will be a problem because we need to copy the unused memory from GPU to host(or from host to GPU) when GPU memory oversubscription.

So our idea is to free the unused memory every iteration, and pytorch provide a API :
* torch.cuda.empty_cache(): Releases all unoccupied cached memory currently held by the caching allocator so that those can be used in other GPU application and visible in nvidia-smi.
We just use this API in our library to help the client to empty the cache after each iteration. 


## Build

Basically all components can be built with the following command:

```
make [CUDA_PATH=/path/to/cuda/installation] [PREFIX=/place/to/install] [DEBUG=1] [TORCH_INCLUDE_PATH=/path/to/torch/installation]
```

This command will install the built binaries in `$(PREFIX)/bin` and `$(PREFIX)/lib`. Default value for `PREFIX` is `$(pwd)/..`.

Adding `DEBUG=1` in above command will make hook library and executables outputs more scheduling details.

## Usage

### resource configuration file format

First line contains an integer *N*, indicating there are *N* clients.

The following *N* lines are of the format:
```
[ID] [REQUEST] [LIMIT] [GPU_MEM]
```
* `ID`: name of the client (ASCII string less than 63 characters). We use this name as identifier of client, so this name must be unique.
* `REQUEST`: minimum required ratio of GPU usage time (between 0 and 1).
* `LIMIT`: maximum allowed ratio of GPU usage time (between 0 and 1).
* `GPU_MEM`: maximum allowed GPU memory usage (in *bytes*).

Changes to this file will be monitored by `gem-schd`. After each change, scheduler will read this file again and update settings. (\*Note that client must restart to get new memory limit)

### Run

We provide two Python scripts under `tools/` for launching *scheduling system* (`launch-backend.py`) (launches *scheduler* and *pod managers*) and applications (`launch-command.py`).

By default scheduler uses port `50051`, and pod managers use ports starting from `50052`  (`50052`, `50053`, ...).

For more details, refer to those scripts and source code.

### Example

We provide a example to demonstrate how to use this library and its use case and benefits.
#### Testbed:
- GPU : GeForce RTX 2080 Ti
- NVIDIA driver : 440.118.02
- CUDA : 10.2
- PyTorch : 1.4.0

#### Application:
- [detectron2](https://github.com/facebookresearch/detectron2) :
We use detectron2's inference benchmark as our test application. And to show the benefits of GPU-sharing, we add sleep() between each iteration.(GPU-sharing benefits when multiple low gpu utiliation jobs running on the same GPU, so we add sleep() to decrease the gpu utilization.) And we use faster_rcnn_R_101_C4_3x as the model, the inference benchmark will occupy approximately 4.6 GB of the GPU memory. We launch 5 inference benchmark at the same time, each job has 50 iterations, and the inter-arrival time is 2 seconds , it will cause GPU memory oversubscription.( 4.6*5 = 23 > 11 GB (2080 Ti) )  

#### Running: 
Launches *scheduler* and *pod managers* first:
```
python launch-backend.py /path/to/gemini_um/installation/bin/gem-schd /path/to/gemini_um/installation/bin/gem-pmgr # launches *scheduler* and *pod managers*
```
Launches application : 
```
python launch-command.py --name = client1 --port=50052 -- python tools/benchmark.py --config-file configs/COCO-Detection/faster_rcnn_R_101_C4_3x.yaml --task eval
python launch-command.py --name = client2 --port=50053 -- python tools/benchmark.py --config-file configs/COCO-Detection/faster_rcnn_R_101_C4_3x.yaml --task eval
          .
          .
python launch-command.py --name = client5 --port=50056 -- python tools/benchmark.py --config-file configs/COCO-Detection/faster_rcnn_R_101_C4_3x.yaml --task eval
```
#### Experiment result:
We sum up each job's throughput to get aggregate throughput :
- no unified memory optimization : 0.454 iteration/second
- with unified memory optimization : 0.962 iteration/second

With torch.cuda.empty_cache(), we get a significant performance improvement due to release the unused memory which can reduce the data movement time. And we use nvml to observe the GPU memory usage. 
The average GPU memory usage:
- no unified memory optimization : 4.6*5 = 23 GB (memory oversubscription)
- with unified memory optimization : 7.65 GB 

From the observation of average GPU memory usage, we can know the unified memory optimization not only can improve the performance, it also can afford more jobs running on the same GPU.

### Webhook

We also provide a webhook for k8s system. The mutation webhook will mount the directory which contains the hook library for the pods that use GPU. And then GPU pods can do GPU sharing through our hook library. Our webhook is based on a webhook example.(https://github.com/cnych/admission-webhook-example) 

### How to launch the webhook

You can follow the steps in /webhook/README.md.

## Contributors

[jim90247](https://github.com/jim90247)
[eee4017](https://github.com/eee4017)
[ncy9371](https://github.com/ncy9371)


