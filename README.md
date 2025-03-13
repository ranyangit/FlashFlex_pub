# HexiScale: Accommodating Large Language Model Training over Heterogeneous Environment [[paper](https://arxiv.org/pdf/2409.01143)]

HexiScale is a roubost system that accomodates **hybrid 3D parallelism** for Llama-2 model pretraining. 
Key features include:
- Versitile support for hybrid pipeline parallelism, tensor parallelism and data parallelism.
- Each transformer layer could have different tensor model parallelism size.
- Each pipeline could have different batch size, and each stage could have arbitrary number of transformer layers.

----------

This project was made possible thanks to a collaboration with

<img src="https://github.com/user-attachments/assets/5ebf23cd-e95e-46c3-abbb-f2f2d60611f9" width="466" height="150" alt="hkust_pku">


----------

## Content

- [Environment](#environment)
- [Asymmetric Parallel Group Support in HexiScale](#asymmetric-parallel-group-support-in-HexiScale)
  - [Parallelisms](#parallelisms)
  - [Asymmetric Parallel Group Support](#asymmetric-parallel-group-support)
- [Launch Processes](#launch-processes)
  - [Launching Processes with Torchrun](#launch-processes-with-torchrun)
  - [Launch Processes Independently](#launch-processes-independently)
- [Hierarchical Graph Partitioning Algorithm](#hierarchical-graph-partitioning-algorithm)
- [Performance Results](#performance-results)

## Environment

HexiScale is well tested on docker image `nvcr.io/nvidia/pytorch:24.02-py3`, which contains `torch==2.3.0` and `flash-attn==2.4.2`, with the utilization of CUDA version 12.3. It would be easy to build the environment by the provided `Dockerfile`.

## Asymmetric Parallel Group Support in HexiScale

### Parallelisms

3D parallelism are used to distribute the workload of pretraining Llama-2 models.

- **Tensor Model Parallelism** splits the transformer layers' tensors across different GPUs.
- **Pipeline Parallelism** divides the model into different stages, each stage holds a certain number of transformer layers and is processed on a different set of GPUs.
- **Data Parallelism** replicates the model and splits the batch size. Gradients are synchronized across different data parallel groups.

### Asymmetric Parallel Group Support

HexiScale introduces a novel approach with its Asymmetric Parallel Group Support, driven by some critical parameters: 

- `--hetero_configs`: This parameter is a 2-dimensional python list, it tells the overall pipeline layouts of given GPUs.
- `--layer_partitions`: This parameter is also a 2-dimensional python list, which describes how to split transformer layers for each pipeline and each stage.
- `--global_bsz_size`: This parameter accepts a series of numbers, and they will be assigned as each pipeline's batch size.
- `--chunks`: This parameter accepts a series of numbers, and they decides number of micro-batchs of each pipeline.

## Launch Processes

### Launch Processes with Torchrun

When all the machines have the same number of GPUs, launching by `torchrun` is a good choice. It is a standard way to launch the distributed system in this case. An example could be seen in `scripts/train.sh`. 

### Launch Processes Independently

In the case when different machines have different number of processes to launch, the only thing to do is to edit the `scripts/generate_hetero_scripts.sh`. The critical arguments are introduced below.

The rules of deciding `--hetero_configs` and `--layer_partitions` are the same as above. 

The `--devices_id` argument is a three dimensional list, it describes which devices will be used for each stage of each pipeline. 

In principal, `--global_batch_size` and `--micro_batch_num` should have the same length as hetero strategies. However, if they have smaller length, they will be padded to the same length; otherwise, the lengthy part will be truncted.

The `--master_addr` argument works similar to `torchrun`, it is necessary to set it according to which machine dose `RANK=0` lie on.

The padding rule is very simple, for the pipelines that haven't been assigned with a batch size number, the system will assume they have the same batch size number as the last seen batch size number. 

One of the examples is as follows:

```bash
python3 llama/generate_hetero_scripts.py \
--retain-run-file \
--model-size llama-7b \
--current_device 0 \
--master_addr 100.65.193.86 \
--master_port 9998 \
--layer-num 4 \
--micro_batch_num 1  \
--global_batch_size 1  \
--hetero_configs "[[1, 2, 1],]" \
--layer_partitions "[[1, 2, 1]]"  \
--devices_id "[[[0],  [0, 0], [0]],]" \
--accum-iter 2 \
--run-iter 10 \
```

After editting the script, run the following commands to launch the program:

```bash
bash scripts/generate_hetero_scripts.sh
bash scripts/batch_run_scripts.sh
```

## Hierarchical Graph Partitioning Algorithm

**HexiScale** formulates the scheduling problem of allocating the LLM training computation over a set of heterogeneous GPU devices as a constrained optimization problem. To solve this problem efficiently, **HexiScale** proposes a two-phase optimization approach that employs a graph partitioning algorithm to effectively coordinate parallel strategies for the given set of devices. An example of graph partitioning over heterogeneous GPUs is demonstrated below.

![graph_partitioning_algorithm](https://github.com/user-attachments/assets/bb536801-8cf9-4323-a54e-13be4b2fd3d3)

The detailed explanation of the algorithm and its workings is provided in the [paper](https://arxiv.org/pdf/2409.01143).

## Performance Results
Experimental results provide a detailed comparison in gap of MFU in training **Llama-2 (7B), Llama-2 (13B), Llama (30B)**. 

Homogeneous data center with 2x8xA100-PCIe and TCP connections training by **Megatron** achieved 38.58% and 39.46% for **Llama-2 (7B), Llama-2 (13B)**,  respectively.

Homogeneous data center with 4x8xA100-PCIe and TCP connections training by **Megatron** achieved 27.79% **Llama (30B)**.

**HexiScale** under heterogeneous settings with the *same* total **FLOPS** can achieve MFU in small gap:

Trining with 1x8x3080Ti-12G, 1x8x3090-24G, and 3x8x4090-24G achieved optimal MFU of 31.19% and 27.23% for **Llama-2 (7B), Llama-2 (13B)**,  respectively.

Trining with 1x8x3080Ti-12G, 1x8x3090-24G, 1x8x4090-24G, and 1x8xNVIDIA A100 NVLINK-80G achieved optimal MFU of 33.47% and 31.39% for **Llama-2 (7B), Llama-2 (13B)**,  respectively.

Trining with 1x8x3090-24G, 2x4x4090-24G, 4x8x4090-24G, and 1x8xNVIDIA A100 NVLINK-80G achieved optimal MFU of 27.49% for **Llama (30B)**.

The smallest gap in MFU compared to the homogeneous data center is 0.3%.
