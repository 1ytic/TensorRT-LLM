# DeepSeek‑V3

This guide walks you through the complete process of running the DeepSeek‑v3 model using NVIDIA's TensorRT-LLM framework with the PyTorch backend. It covers everything from downloading the model weights, preparing the dataset and configuration files, to running the throughput benchmark.

> [!NOTE]
> This guide assumes you have access to the required hardware (with sufficient GPU memory) and that you replace placeholder values (e.g. `<YOUR_MODEL_DIR>`) with the appropriate paths. Please refer to [this guide](https://nvidia.github.io/TensorRT-LLM/installation/build-from-source-linux.html) for how to build TensorRT-LLM from source and docker image.


## Table of Contents

- [DeepSeek‑V3](#deepseekv3)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [Hardware](#hardware)
  - [Downloading the Model Weights](#downloading-the-model-weights)
  - [Quick Start](#quick-start)
    - [Multi-Token Prediction (MTP)](#multi-token-prediction-mtp)
    - [Run evaluation on GPQA dataset](#run-evaluation-on-gpqa-dataset)
  - [Preparing the Dataset \& Configuration for Benchmark](#preparing-the-dataset--configuration-for-benchmark)
    - [Build the Dataset](#build-the-dataset)
    - [Create Configuration Files](#create-configuration-files)
  - [Running the Benchmark](#running-the-benchmark)
    - [Parameter Overview](#parameter-overview)
  - [Serving](#serving)
  - [Multi-node](#multi-node)
    - [mpirun](#mpirun)
    - [Slurm](#slurm)
    - [Advanced Features](#advanced-features)
    - [FlashMLA](#flashmla)
    - [DeepGEMM](#deepgemm)
  - [Notes and Troubleshooting](#notes-and-troubleshooting)


## Overview

DeepSeek‑v3 is a high‑capacity language model that can be executed using NVIDIA's TensorRT-LLM framework with a PyTorch backend. This guide details a benchmark recipe where you will:

- Download the model weights.
- Build a test dataset.
- Configure backend options.
- Run a performance benchmark using `trtllm-bench`.


## Hardware
DeepSeek-v3 has 671B parameters which needs about 671GB GPU memory. 8\*H100 (640GB) is not enough to accommodate the weights. The following steps have been tested on 8\*H20 141GB, we will test on 8*H20 96GB in the future.

DeepSeek-v3 is trained natively with FP8 precision, we only provide FP8 solution in TensorRT-LLM at this moment. Ampere architecture (SM80 & SM86) is not supported.


## Downloading the Model Weights

DeepSeek‑v3 model weights are available on [Hugging Face](https://huggingface.co/deepseek-ai/DeepSeek-V3). To download the weights, execute the following commands (replace `<YOUR_MODEL_DIR>` with the target directory where you want the weights stored):

```bash
git lfs install
git clone https://huggingface.co/deepseek-ai/DeepSeek-V3 <YOUR_MODEL_DIR>
```


## Quick Start

To quickly run DeepSeek-V3, [examples/pytorch/quickstart_advanced.py](../pytorch/quickstart_advanced.py):

```bash
cd examples/pytorch
python quickstart_advanced.py --model_dir <YOUR_MODEL_DIR> --tp_size 8
```

The model will be run by PyTorch backend and generate outputs like:
```
Processed requests: 100%|███████████████████████████████████████████████████████████████████████████████████████████████████████████████| 4/4 [00:02<00:00,  1.66it/s]
Prompt: 'Hello, my name is', Generated text: " Dr. Anadale. I teach philosophy at Mount St. Mary's University in Emmitsburg, Maryland. This is the first in a series of videos"
Prompt: 'The president of the United States is', Generated text: ' the head of state and head of government of the United States, indirectly elected to a four-year term via the Electoral College. The officeholder leads the executive branch'
Prompt: 'The capital of France is', Generated text: ' Paris. Paris is one of the most famous and iconic cities in the world, known for its rich history, culture, art, fashion, and cuisine. It'
Prompt: 'The future of AI is', Generated text: ' a topic of great interest and speculation. While it is impossible to predict the future with certainty, there are several trends and possibilities that experts have discussed regarding the future'
```

### Multi-Token Prediction (MTP)
To run with MTP, use [examples/pytorch/quickstart_advanced.py](../pytorch/quickstart_advanced.py).
```bash
cd examples/pytorch
python quickstart_advanced.py --model_dir <YOUR_MODEL_DIR> --spec_decode_algo MTP --spec_decode_nextn N
```

`N` is the number of MTP modules. When `N` is equal to `0`, which means that MTP is not used (default). When `N` is greater than `0`, which means that `N` MTP modules are enabled. In the current implementation, the weight of each MTP module is shared.

### Run evaluation on GPQA dataset
Download the dataset first
1. Sign up a huggingface account and request the access to the gpqa dataset: https://huggingface.co/datasets/Idavidrein/gpqa
2. Download the csv file from https://huggingface.co/datasets/Idavidrein/gpqa/blob/main/gpqa_diamond.csv

Evaluate on GPQA dataset.
```
python examples/gpqa_llmapi.py \
  --hf_model_dir <YOUR_MODEL_DIR> \
  --data_dir <DATASET_PATH> \
  --tp_size 8 \
  --use_cuda_graph \
  --enable_overlap_scheduler \
  --concurrency 32 \
  --batch_size 32 \
  --max_num_tokens 4096
```

## Preparing the Dataset & Configuration for Benchmark

### Build the Dataset

Generate a synthetic dataset using the provided script. This dataset simulates token sequences required for benchmarking:

```bash
python benchmarks/cpp/prepare_dataset.py \
  --tokenizer=deepseek-ai/DeepSeek-V3 \
  --stdout token-norm-dist \
  --num-requests=8192 \
  --input-mean=1000 \
  --output-mean=1000 \
  --input-stdev=0 \
  --output-stdev=0 > /workspace/dataset.txt
```

This command writes the dataset to `/workspace/dataset.txt`.

### Create Configuration Files

1. **Backend Configuration:**
   Enable attention data‑parallelism and overlap scheduler:

   ```bash
   echo -e "enable_attention_dp: true\npytorch_backend_config:\n  enable_overlap_scheduler: true" > extra-llm-api-config.yml
   ```

   If you are running with low concurrency (e.g. concurrency <= 128), we suggest you to enable cuda graph and disable attention_dp for better performances. Please note that `cuda_graph_max_batch_size` should be no less than the concurrency set in the benchmark command.

   ```bash
   echo -e "enable_attention_dp: false\npytorch_backend_config:\n  enable_overlap_scheduler: true\n  use_cuda_graph: true\n  cuda_graph_max_batch_size: 128" > extra-llm-api-config.yml
   ```

2. **Quantization Configuration:**
   Configure the quantization settings for the model:

   ```bash
   echo -e "{\"quantization\": {\"quant_algo\": \"FP8_BLOCK_SCALES\", \"kv_cache_quant_algo\": null}}" > <YOUR_MODEL_DIR>/hf_quant_config.json
   ```

> [!TIP]
> Ensure that the quotes and formatting in the configuration files are correct to avoid issues during runtime.


## Running the Benchmark

With the model weights downloaded, the dataset prepared, and the configuration files in place, run the benchmark using the following command:

```bash
trtllm-bench \
  --model deepseek-ai/DeepSeek-V3 \
  --model_path  <YOUR_MODEL_DIR> \
  throughput \
  --backend pytorch \
  --max_batch_size 161 \
  --max_num_tokens 1160 \
  --dataset /workspace/dataset.txt \
  --tp 8 \
  --ep 4 \
  --pp 1 \
  --concurrency 1024 \
  --streaming \
  --kv_cache_free_gpu_mem_fraction 0.95 \
  --extra_llm_api_options ./extra-llm-api-config.yml 2>&1 | tee /workspace/trt_bench.log
```

### Parameter Overview

- `--model`: Specifies the model identifier.
- `--model_path`: Path to the directory where model weights are stored.
- `throughput`: Sets benchmark mode to measure throughput.
- `--backend pytorch`: Selects the PyTorch backend.
- `--max_batch_size` & `--max_num_tokens`: Define runtime batch size and token length.
- `--tp 8`: Configures tensor parallelism.
- `--ep 4`: Configures expert parallelism.
- `--kv_cache_free_gpu_mem_fraction`: Allocates a fraction of GPU memory for KV cache management.
- `--extra_llm_api_options`: Provides additional configuration options from the specified file.

Benchmark logs are saved to `/workspace/trt_bench.log`.


## Serving

To serve the model using `trtllm-serve`:
```bash
trtllm-serve \
  deepseek-ai/DeepSeek-V3 \
  --host localhost \
  --port 8000 \
  --backend pytorch \
  --max_batch_size 161 \
  --max_num_tokens 1160 \
  --tp_size 8 \
  --ep_size 4 \
  --pp_size 1 \
  --kv_cache_free_gpu_memory_fraction 0.95 \
  --extra_llm_api_options ./extra-llm-api-config.yml
```

To query the server, you can start with a `curl` command:
```bash
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
      "model": "deepseek-ai/DeepSeek-V3",
      "prompt": "Where is New York?",
      "max_tokens": 16,
      "temperature": 0
  }'
```

For DeepSeek-R1, use the model name `deepseek-ai/DeepSeek-R1`.

## Multi-node
TensorRT-LLM supports multi-node inference. You can use mpirun or Slurm to launch multi-node jobs. We will use two nodes for this example.

### mpirun
mpirun requires each node to have passwordless ssh access to the other node. We need to setup the environment inside the docker container. Run the container with host network and mount the current directory as well as model directory to the container.

```bash
# use host network
IMAGE=<YOUR_IMAGE>
NAME=test_2node_docker
# host1
docker run -it --name ${NAME}_host1 --ipc=host --gpus=all --network host --privileged --ulimit memlock=-1 --ulimit stack=67108864 -v ${PWD}:/workspace -v <YOUR_MODEL_DIR>:/models/DeepSeek-V3 -w /workspace ${IMAGE}
# host2
docker run -it --name ${NAME}_host2 --ipc=host --gpus=all --network host --privileged --ulimit memlock=-1 --ulimit stack=67108864 -v ${PWD}:/workspace -v <YOUR_MODEL_DIR>:/models/DeepSeek-V3 -w /workspace ${IMAGE}
```

Set up ssh inside the container

```bash
apt-get update && apt-get install -y openssh-server

# modify /etc/ssh/sshd_config
PermitRootLogin yes
PubkeyAuthentication yes
# modify /etc/ssh/sshd_config, change default port 22 to another unused port
port 2233

# modify /etc/ssh
```

Generate ssh key on host1 and copy to host2, vice versa.

```bash
# on host1
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<HOST2>
# on host2
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<HOST1>

# restart ssh service on host1 and host2
service ssh restart # or
/etc/init.d/ssh restart # or
systemctl restart ssh
```

You can use the following example to test mpi communication between two nodes:
```cpp
#include <mpi.h>
#include <stdio.h>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);

    int world_rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);

    int world_size;
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);

    printf("Hello world from rank %d out of %d processors\n", world_rank, world_size);

    MPI_Finalize();
    return 0;
}
```

Compile and run the hello world program on two nodes:
```bash
mpicc -o mpi_hello_world mpi_hello_world.c
mpirun -np 2 -H <HOST1>:1,<HOST2>:1 -mca plm_rsh_args "-p 2233" ./mpi_hello_world
# Hello world from rank 11 out of 16 processors
# Hello world from rank 13 out of 16 processors
# Hello world from rank 12 out of 16 processors
# Hello world from rank 15 out of 16 processors
# Hello world from rank 14 out of 16 processors
# Hello world from rank 10 out of 16 processors
# Hello world from rank 9 out of 16 processors
# Hello world from rank 8 out of 16 processors
# Hello world from rank 5 out of 16 processors
# Hello world from rank 2 out of 16 processors
# Hello world from rank 4 out of 16 processors
# Hello world from rank 6 out of 16 processors
# Hello world from rank 3 out of 16 processors
# Hello world from rank 1 out of 16 processors
# Hello world from rank 7 out of 16 processors
# Hello world from rank 0 out of 16 processors
```

Prepare the dataset and configuration files on two nodes as mentioned above. Then you can run the benchmark on two nodes using mpirun:

```bash
mpirun \
--output-filename bench_log_2node_ep8_tp16_attndp_on_sl1000 \
-H <HOST1>:8,<HOST2>:8 \
-mca plm_rsh_args "-p 2233" \
--allow-run-as-root -n 16 \
trtllm-llmapi-launch trtllm-bench --model deepseek-ai/DeepSeek-V3 --model_path /models/DeepSeek-V3 throughput --backend pytorch --max_batch_size 161 --max_num_tokens 1160 --dataset /workspace/tensorrt_llm/dataset_isl1000.txt --tp 16 --ep 8 --kv_cache_free_gpu_mem_fraction 0.95 --extra_llm_api_options /workspace/tensorrt_llm/extra-llm-api-config.yml --concurrency 4096 --streaming
```

### Slurm
```bash
  srun -N 2 -w [NODES] \
  --output=benchmark_2node.log \
  --ntasks 16 --ntasks-per-node=8 \
  --mpi=pmix --gres=gpu:8 \
  --container-image=<CONTAINER_IMG> \
  --container-mounts=/workspace:/workspace \
  --container-workdir /workspace \
  bash -c "trtllm-llmapi-launch trtllm-bench --model deepseek-ai/DeepSeek-V3 --model_path <YOUR_MODEL_DIR> throughput --backend pytorch --max_batch_size 161 --max_num_tokens 1160 --dataset /workspace/dataset.txt --tp 16 --ep 4 --kv_cache_free_gpu_mem_fraction 0.95 --extra_llm_api_options ./extra-llm-api-config.yml"
```

### Advanced Features
### FlashMLA
TensorRT-LLM has already integrated FlashMLA in the PyTorch backend. It is enabled automatically when running DeepSeek-V3/R1.

### DeepGEMM
TensorRT-LLM also supports DeepGEMM for DeepSeek-V3/R1, which provides significant e2e performance boost on Hopper GPUs. DeepGEMM is enabled by an environment variable `TRTLLM_DG_ENABLED`:

DeepGEMM-related behavior can be controlled by the following environment variables:

| Environment Variable | Description |
| ----------------------------- | ----------- |
| `TRTLLM_DG_ENABLED` | When set to `1`, enable DeepGEMM. |
| `TRTLLM_DG_JIT_DEBUG` | When set to `1`, enable JIT debugging. |
| `TRTLLM_DG_JIT_USE_NVCC` | When set to `1`, use NVCC instead of NVRTC to compile the kernel, which has slightly better performance but requires CUDA Toolkit (>=12.3) and longer compilation time.|
| `TRTLLM_DG_JIT_DUMP_CUBIN` | When set to `1`, dump the cubin file. This is only effective with NVRTC since NVCC will always dump the cubin file. NVRTC-based JIT will store the generated kernels in memory by default. If you want to persist the kernels across multiple runs, you can either use this variable or use NVCC. |

```bash
#single-node
TRTLLM_DG_ENABLED=1 \
trtllm-bench \
      --model deepseek-ai/DeepSeek-V3 \
      --model_path /models/DeepSeek-V3 \
      throughput \
      --backend pytorch \
      --max_batch_size ${MAX_BATCH_SIZE} \
      --max_num_tokens ${MAX_NUM_TOKENS} \
      --dataset dataset.txt \
      --tp 8 \
      --ep 8 \
      --kv_cache_free_gpu_mem_fraction 0.9 \
      --extra_llm_api_options /workspace/extra-llm-api-config.yml \
      --concurrency ${CONCURRENCY} \
      --num_requests ${NUM_REQUESTS} \
      --streaming \
      --report_json "${OUTPUT_FILENAME}.json"

# multi-node
mpirun -H <HOST1>:8,<HOST2>:8 \
      -n 16 \
      -x "TRTLLM_DG_ENABLED=1" \
      -x "CUDA_HOME=/usr/local/cuda" \
      trtllm-llmapi-launch trtllm-bench \
      --model deepseek-ai/DeepSeek-V3 \
      --model_path /models/DeepSeek-V3 \
      throughput \
      --backend pytorch \
      --max_batch_size ${MAX_BATCH_SIZE} \
      --max_num_tokens ${MAX_NUM_TOKENS} \
      --dataset dataset.txt \
      --tp 16 \
      --ep 16 \
      --kv_cache_free_gpu_mem_fraction 0.9 \
      --extra_llm_api_options /workspace/extra-llm-api-config.yml \
      --concurrency ${CONCURRENCY} \
      --num_requests ${NUM_REQUESTS} \
      --streaming \
      --report_json "${OUTPUT_FILENAME}.json"
```

## Notes and Troubleshooting

- **Model Directory:** Update `<YOUR_MODEL_DIR>` with the actual path where the model weights reside.
- **GPU Memory:** Adjust `--max_batch_size` and `--max_num_tokens` if you encounter out-of-memory errors.
- **Logs:** Check `/workspace/trt_bench.log` for detailed performance information and troubleshooting messages.
- **Configuration Files:** Verify that the configuration files are correctly formatted to avoid runtime issues.


By following these steps, you should be able to successfully run the DeepSeek‑v3 benchmark using TensorRT-LLM with the PyTorch backend.
