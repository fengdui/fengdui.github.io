---
title: "gkd llm知识蒸馏记录"
date: "2025-01-16"
tags: ["人工智能"]
ShowToc: false
TocOpen: false
---
```
NPROC_PER_NODE=7 \
PYTORCH_CUDA_ALLOC_CONF='expandable_segments:True' \
CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6 \
swift rlhf \
    --rlhf_type gkd \
    --model /mnt/model/Qwen3-VL-8B-Instruct \
    --teacher_model /mnt/model/Qwen3-VL-235B-A22B-Instruct \
    --train_type full \
    --dataset /mnt/data/dataset.jsonl \
    --seq_kd false \
    --lmbda 1 \
    --beta 1 \
    --torch_dtype bfloat16 \
    --num_train_epochs 1 \
    --per_device_train_batch_size 4 \
    --learning_rate 1e-5 \
    --gradient_accumulation_steps 2 \
    --save_steps 1000 \
    --save_total_limit 2 \
    --logging_steps 1 \
    --max_length 16000 \
    --max_completion_length 8192 \
    --output_dir output \
    --warmup_ratio 0.05 \
    --save_only_model true \
    --dataloader_num_workers 64 \
    --dataset_num_proc 4 \
    --deepspeed zero2 \
    --teacher_deepspeed zero3 \
    --attn_impl flash_attn \
    --use_vllm true \
    --vllm_mode server \
    --vllm_server_host 127.0.0.1 \
    --vllm_server_port 8000 \
    --output_dir /mnt/data/trainingmodel/ \
```
```
CUDA_VISIBLE_DEVICES=7 \
swift rollout \
    --model /mnt/model/Qwen3-VL-8B-Instruct \
    --vllm_max_model_len 16000
```