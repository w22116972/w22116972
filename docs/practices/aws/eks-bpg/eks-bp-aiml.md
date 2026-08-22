# Amazon EKS Best Practices：AI/ML

在 EKS 上執行 AI/ML 平台時，必須協調稀缺的 accelerator、大型 model artifacts、高吞吐量 networking、長時間執行的 jobs，以及 inference latency。應從 scheduling、data loading、model quality 到使用者可見的 request performance，最佳化完整流程。

> 最後檢視：2026-08-20

## Compute 與 scheduling

- 使用支援的 well-known labels 與 device plugins 申請 accelerator；以 taints 與 tolerations 隔離 accelerated nodes，避免一般 workloads 消耗昂貴容量。
- 允許多種相容的 instance families 與 sizes，依 workload 可預測性、capacity assurance、startup time 與 interruption sensitivity 選擇 Karpenter 或 static nodes。
  - Karpenter 適合需求變化大、可接受動態 provision 的 workload；static nodes 適合必須預留容量或啟動時間不可預測的 critical workload。
- 對可容錯 jobs 使用 Spot，並為長時間 training 建立 checkpoint，使工作可在中斷後恢復；嚴格 training windows 則使用 On-Demand、reservations 或 Capacity Blocks。
- 依 isolation、utilization、performance 與 operational requirements，選擇 full GPU allocation、time-slicing、Multi-Instance GPU (MIG)、fractional allocation、MPS 或 Dynamic Resource Allocation。
  - Full GPU isolation 最強但可能浪費容量；sharing 可提高 utilization，代價是 workload 之間可能競爭 memory、compute 或 failure domain。
  - 這些方案不是無條件互換：採用前須確認 EKS/Kubernetes version、accelerator hardware、device plugin 或 vendor DRA driver 是否支援；DRA 在 EKS 1.33 可用，而 core DRA 從 Kubernetes 1.34 起成為 GA。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/best-practices/aiml-compute.html) [Kubernetes 官方文件](https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/)
- 自動偵測並修復 node health；對 interruption-sensitive distributed jobs 限制 consolidation，使用 priority 與 preemption，並以 `ttlSecondsAfterFinished` 清理已完成的 Jobs。

## CPU 與 accelerators 的選擇

- 在選擇 CPU、GPU 或 Trainium 前，先以代表性 models、batch sizes、sequence lengths、concurrency 與 quality targets 進行 benchmark。
- CPU inference 適合小型或 quantized models、burstable endpoints、dense model farms，以及重視容量與簡潔性的 agentic pipeline stages。
- 使用 quantization 與高效率 serving runtimes，依實測 memory 與 core 需求進行 bin-pack，並確認最佳化不會違反 model-quality thresholds。
- 依 queue depth、concurrency、request latency 或 token demand 擴展 inference；單獨依 CPU utilization 通常反應太慢，也未必能描述 saturation。
  - 例如 GPU 已滿載或 requests 已在 queue 中等待時，CPU 仍可能偏低；此時只看 CPU 會延後 scale-out。
  - 可使用 KEDA 將 inference requests、queue depth 或 token throughput 轉成 scaling signal，並設定適當 cooldown period，避免低流量期間頻繁 scale in/out。[AWS 官方文件](https://docs.aws.amazon.com/eks/latest/best-practices/aiml-compute.html)

## Networking、storage 與 startup

- 對大量 inter-node communication 的 distributed jobs 使用高頻寬 instances 與 Elastic Fabric Adapter；使用 Spot 前先規劃 topology 與 placement。
- 估算大型 accelerated instances 與快速 scale-out 的 IP 消耗。
- 透過 CSI driver 或適當的 shared cache 提供 model artifacts，依 access pattern、sharing、throughput、latency、persistence 與 cost 選擇 FSx for OpenZFS、EFS、S3 Express One Zone、EBS 或 node-local NVMe。
  - 先確認 model 是由多個 pods 共享、每次啟動重新下載，還是需要 node-local cache；這些 access patterns 會直接影響 cold-start 與 storage 選擇。
- 減少 image size 與 layers；使用 local caches、pre-pulls、SOCI、EBS snapshots 與足夠的 image-pull bandwidth 改善 cold starts。
- 若能縮短 build 與 deployment time，將 immutable images 與大型且經常更新的 model artifacts 分離。

## Security 與 observability

- 加密 model artifacts 與 datasets；在 compliance 要求時使用 KMS 加密 S3 data。每個 workload 使用 least-privileged identity，限制 proprietary models 與 training data 的存取。
- 除 node、storage 與 network signals 外，監控 accelerator allocation、utilization、power、temperature、memory、errors 與 interconnect health。
- Training 與 fine-tuning 追蹤 loss、learning rate、step time、throughput、checkpoint progress 與 failed workers。
- Online inference 追蹤 queue time、time to first token、request duration、tokens per second、batch size、errors 與 tail latency，並以這些使用者可見 metrics 驅動 autoscaling 與 capacity planning。
- 在 quantization、runtime changes、model updates 與 infrastructure optimization 後持續評估 model quality。
  - Infrastructure metrics 變好不代表結果仍正確；應以相同 evaluation dataset 比較 accuracy、quality 與 safety thresholds。

## Review checklist

- Instance、capacity 與 sharing 選擇均有 workload benchmarks 支持。
- Training 可在 node loss 後恢復，不必重啟所有有價值的工作。
- Model storage 與 image delivery 符合 cold-start 與 throughput 目標。
- Autoscaling 適當使用 queue、concurrency、latency 或 token signals。
- Infrastructure metrics 與 model-quality evaluation 一併檢視。

---
# Amazon EKS Best Practices: AI/ML

AI/ML platforms on EKS must coordinate scarce accelerators, large model
artifacts, high-throughput networking, long-running jobs, and inference latency.
Optimize the complete path from scheduling and data loading to model quality and
user-visible request performance.

## Compute and scheduling

- Request accelerators with supported well-known labels and device plugins.
  Isolate accelerated nodes with taints and tolerations so general workloads do
  not consume expensive capacity.
- Allow multiple compatible instance families and sizes. Use Karpenter or static
  nodes according to workload predictability, capacity assurance, startup time,
  and interruption sensitivity.
- Use Spot for fault-tolerant jobs and checkpoint long-running training so work
  can resume after interruption. Use On-Demand, reservations, Capacity Blocks,
  or other assured capacity for strict training windows.
- Select full GPU allocation, time-slicing, Multi-Instance GPU (MIG), fractional
  allocation, MPS, or Dynamic Resource Allocation from isolation, utilization,
  performance, and operational requirements.
- Automate node health detection and recovery. Disable or constrain consolidation
  for interruption-sensitive distributed jobs, use priority and preemption, and
  remove completed Jobs with `ttlSecondsAfterFinished`.

## Decide between CPU and accelerators

- Benchmark representative models, batch sizes, sequence lengths, concurrency,
  and quality targets before selecting CPU, GPU, or Trainium.
- CPU inference can fit small or quantized models, burstable endpoints, dense
  model farms, and agentic pipeline stages where capacity and simplicity matter.
- Use quantization and efficient serving runtimes, bin-pack from measured memory
  and core needs, and validate that optimization does not violate model-quality
  thresholds.
- Scale inference on queue depth, concurrency, request latency, or token demand;
  CPU utilization alone often reacts too late or does not describe saturation.

## Networking, storage, and startup

- Use higher-bandwidth instances and Elastic Fabric Adapter for distributed jobs
  with heavy inter-node communication. Plan topology and placement before using
  Spot because tightly coupled capacity can be harder to replace.
- Account for IP consumption on large accelerated instances and during rapid
  scale-out.
- Deliver model artifacts through a CSI driver or an appropriate shared cache.
  Select FSx for OpenZFS, EFS, S3 Express One Zone, EBS, or node-local NVMe from
  access pattern, sharing, throughput, latency, persistence, and cost.
- Reduce image size and layers. Improve cold starts with local caches, pre-pulls,
  lazy loading such as SOCI, EBS snapshots, and adequate image-pull bandwidth.
- Separate immutable images from large, frequently updated model artifacts when
  that reduces build and deployment time.

## Security and observability

- Encrypt model artifacts and datasets, including S3 data with KMS where
  compliance requires it. Give each workload a least-privileged identity and
  restrict access to proprietary models and training data.
- Monitor accelerator allocation, utilization, power, temperature, memory,
  errors, and interconnect health alongside node, storage, and network signals.
- For training and fine-tuning, track loss, learning rate, step time, throughput,
  checkpoint progress, and failed workers.
- For online inference, track queue time, time to first token, request duration,
  tokens per second, batch size, errors, and tail latency. Tie autoscaling and
  capacity planning to these user-visible metrics.
- Continuously evaluate model quality after quantization, runtime changes, model
  updates, and infrastructure optimization.

## Review checklist

- Instance, capacity, and sharing choices are backed by workload benchmarks.
- Training can recover from node loss without restarting all useful work.
- Model storage and image delivery meet cold-start and throughput objectives.
- Autoscaling uses queue, concurrency, latency, or token signals as appropriate.
- Infrastructure metrics and model-quality evaluation are reviewed together.
