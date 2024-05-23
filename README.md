## TGI benchmark

TGI benchmark with TP=8 can be reproduced as follow on MI250 and MI300:
```
docker run --rm -it --cap-add=SYS_PTRACE --security-opt seccomp=unconfined \
    --device=/dev/kfd --device=/dev/dri --group-add video --ipc=host --shm-size 256g \
    --net host -v $(pwd)/hf_cache:/data -e HUGGING_FACE_HUB_TOKEN=$HF_READ_TOKEN \
    ghcr.io/huggingface/text-generation-inference:sha-293b8125-rocm \
    --model-id meta-llama/Meta-Llama-3-70B-Instruct --num-shard 8
```

Then, a second shell needs to be open in TGI's server container:
```
docker container ls
docker exec -it container_name /bin/bash
```

From the second shell:
```
huggingface-cli login --token your_hf_read_token

text-generation-benchmark --tokenizer-name meta-llama/Meta-Llama-3-70B-Instruct \
    --sequence-length 2048 --decode-length 128 --warmups 2 --runs 10 \
    -b 1 -b 2 -b 4 -b 8 -b 16 -b 32 -b 64
```

Once the benchmark is finished, one can press Ctrl+C in the benchmark shell and should find a markdown table summarizing prefill and decode latency, as well as throughput.

Note: TGI's tool `text-generation-benchmark` tends to OOM, which does not reflect the real memory limit of the benchmarked GPUs. For reference: https://github.com/huggingface/text-generation-inference/issues/1831, https://github.com/huggingface/text-generation-inference/issues/1286

Note: Once released, we recommend to use the image `ghcr.io/huggingface/text-generation-inference:2.1-rocm` instead of `ghcr.io/huggingface/text-generation-inference:sha-293b8125-rocm`. TGI on ROCm can also be built from source using [this dockerfile](https://github.com/huggingface/text-generation-inference/blob/main/Dockerfile_amd).

### Recommended setup

We recommend setting on the host ([reference](https://huggingface.co/docs/optimum/main/en/amd/amdgpu/perf_hardware#numa-nodes)):
```
sudo sh -c "/usr/bin/echo 0 > /proc/sys/kernel/numa_balancing"
sudo rocm-smi --setperfdeterminism 1900
```

More details: https://github.com/ROCm/triton/wiki/A-script-to-set-program-execution-environment-in-ROCm
