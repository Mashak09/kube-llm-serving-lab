-setup: Docker dektop with 8 cores and about 10.4 GB, minikube v1.37.0, kubernetes v1.34.0, CPU has AVX2 only
-problems i faced: There was a already a minikube cluster running wchich was occupying the CPU's and memory. Fixed with 'minikube delete' and a fresh start.
- Image: vllm/vllm-openai-cpu:v0.26.0-x86_64, digest sha256:8ed97d93c37e8025ca91c4bafaebb03529d55bae68dc40af6609fe129b5dad36.
- Model: Qwen/Qwen3-0.6B (Apache 2.0)
- First run in plain Docker first start about 6 minutes includeing the download , One request produced 20 token in 1.78s (single call, not a benchmark).
- Learned: memory sat near the 4 GiB container limit, vLLM saw 8 cores despite --cpus=4, and the NUMA warnings were harmless.

## 2026-11-16
- What I did: ran vLLM (Qwen3-0.6B, CPU) on minikube with a Deployment, an internal ClusterIP Service and a volume for the model cache, and reached it through kubectl port-forward.
- What broke: drive C: was full (0 GB free), which stalled Docker and left the first image pull stuck for 22 hours. I moved Downloads and Docker's disk to D:. Then the pod was OOMKilled at a 4Gi memory limit; 6Gi works.
- Measurements (single replica, not a benchmark): steady request about 1.2 s for 20 tokens, first request after startup 11.9 s. CPU throttled in 124 of 13483 periods. Memory: rss 4.6 GiB, cache 1.2 GiB, peak reached the 6 GiB limit.
- Learned: two replicas of this setup will not fit into an 8 GB minikube, so the footprint must shrink. The oneDNN kernel falls back to a plain one (probably because the CPU has only AVX2). Always send a warm-up request before measuring.

## 2026-11-17
- What I did: restarted the cluster and found the vllmpod crash-looping after the restart.
- What broke: vLLM refused to start with "VLLM_PORT ... appears to be a URI". Kubernetes injects environment variables for every Service that already exists when a pod starts. My Service is called vllm, so the pod got VLLM_PORT=tcp://10.100.55.87:8000, which vLLM reads as its own port setting. The first pod had started before the Service existed, so it never saw the variable. Fix: enableServiceLinks: false in the pod spec.
- Measured (single replica, not a benchmark): first request after startup 11.5 s, steady requests about 1.2 s.
- Learned: a pod only gets Service environment variables if the Service existed when the pod was created. Read the previous container's logs above the traceback. During a rollout, deploy/NAME can pick the old pod, so use the pod name.
