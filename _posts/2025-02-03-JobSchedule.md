---
title: "任务调度与负载均衡"
description: ""
author: ethan
date: 2025-02-03 08:37:00 +0800
categories: [Linux]
tags: [Linux, Linux Kernel]
pin: false
math: true
mermaid: true
---

## job submit

```
- bstnpu_job_submit:
- bstnpu_job_alloc
  - 创建 job memory
    - 同步模式：创建 job，但是不会深拷贝 job args
    - 异步模式：创建 job，并且深拷贝 job args
  - 创建 task 副本
    - 同步模式：skip
    - 异步模式：利用虚拟内存创建 task 副本，不依赖 dram
- stnpu_job_mm_map: 准备了 tasks's ptr array
- dma_fence_wait_timeout
  - 存在 fence in: wait fence signal 直到超时或者捕获到信号量
  - 没有 fence in: skip
- alloc dma fence
  - 存在 fence out: 从当前 ctx 获取 fence fd
  - 没有 fence out: skip
- bstnpu_job_schedule
  - bstnpu_job_timeout_clean，超时任务清理，清理的都是异步任务 (同步模式下没有考虑这步)
  - job_schedule && job exit
    - 同步模式：schedule 之后紧接 job_wait，根据 wait 结果决定是 job_clean 还是 job_abort
    - 异步模式：schedule 之后直接返回，如果 schedule 失败，则执行 job_abort
```

## job schedule
```
bstnpu_job_schedule
- auto_mask 自动分配，搜寻 task_num 最少的 core，并更新 core_mask
- backend_core.ops 调用 init_cmdbuf 预分配 cmd buffer
- 向 subcore 的 todo_list 添加当前 job
- （接下来的步骤已经和当前 job 没有必要关联了，主要是处理 workqueue 中的内容）
- bstnpu_job_next
  - 当前存在正在处理中的 job，直接返回
  - 当前没有正在处理中的 job，从 todo_list 拿到第一个 job
  - commit 当前 job (Tips: 若 job 依赖多个 core，会等待所有依赖的 core 就绪后才会 commit)
  - backend_core.ops 调用 exec 处理 task start ~ task end
```

## job wait
```
bstnpu_job_wait
- 循环等待 job done 的flag，这里采用了 wait_queue，等待进程会休眠，直到超时或被唤醒
- 根据 last_task 判断是否 commit 成功，如果wait timeout 已经结束，还是没有commit 成功，就会将 job 从 todo_list 中清理掉
- 如果已经 commit 了，但是超时了，（这里的逻辑有些迷惑，感觉没什么处理）
```

## job abort
```
bstnpu_job_abort
- 中断锁获取，（同时要求还没有进入中断handler），判断 subcore 正在执行的 job 是否是当前 job，如果是就要置 NULL (这里没有考虑从 todo_list 清理的情况)
- 如果是超时，则执行 npu soft_reset
- 执行 bstnpu_job_clean，job_alloc 的逆过程 (这里的 fence refc 深度只减了 1)，约定需要用户 close 1 次
```