# DGX Spark에서 Dynamo 프로젝트 격리 실행 플랜

## Context
DGX Spark(Grace Blackwell, ARM64, CUDA 13)로 이전. 이 프로젝트를 다른 프로젝트에 영향 없이 격리 실행하고 싶음.

---

## 실행 플랜: Docker 컨테이너 격리 (추천)

호스트에 아무것도 설치하지 않고, 프로젝트의 기존 빌드 스크립트 활용.

### Step 1: 인프라 서비스 시작 (NATS + etcd)

```bash
cd /home/edr2team/문서/vibe-with-dynamo-inflearn
docker compose -f deploy/docker-compose.yml up -d
```
- NATS(4222), etcd(2379) — 두 이미지 모두 ARM64 지원

### Step 2: Docker 이미지 빌드 (ARM64 + CUDA 13)

```bash
container/build.sh \
  --platform linux/arm64 \
  --framework vllm \
  --cuda-version 13.0 \
  --target local-dev
```
- 결과: `dynamo:latest-vllm-local-dev`
- 빌드 시간: 30~60분+ (Rust 컴파일 포함)

### Step 3: 컨테이너 실행

```bash
container/run.sh \
  --image dynamo:latest-vllm-local-dev \
  --mount-workspace \
  --hf-cache /home/edr2team/.cache/huggingface \
  -it \
  -- bash
```
- `--mount-workspace`: 프로젝트를 `/workspace`에 마운트
- `--hf-cache`: 호스트 모델 캐시 공유 (재다운로드 방지)
- `--network host` (기본): NATS/etcd 접근 가능
- `--gpus all`, `--shm-size=10G`: GPU 및 공유 메모리

### Step 4: 컨테이너 내부에서 추론 실행

```bash
# Frontend (OpenAI 호환 API)
python -m dynamo.frontend --http-port 8000 &

# vLLM Worker (작은 모델로 테스트)
DYN_SYSTEM_PORT=8081 python -m dynamo.vllm \
  --model Qwen/Qwen3-0.6B \
  --gpu-memory-utilization 0.20 \
  --enforce-eager \
  --no-enable-prefix-caching \
  --max-num-seqs 64 &
```

### Step 5: 검증

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen3-0.6B","messages":[{"role":"user","content":"Hello"}]}'
```

### Step 6: 정리

```bash
# 컨테이너 종료: Ctrl+D 또는 exit
docker compose -f deploy/docker-compose.yml down
# 디스크 회수 (선택): docker rmi dynamo:latest-vllm dynamo:latest-vllm-local-dev
```

---

## 격리 보장
- 모든 Python 패키지, Rust 툴체인, CUDA 라이브러리 → 컨테이너 내부
- 호스트 시스템 영향: **제로** (Docker 이미지 저장 공간만 사용)
- HuggingFace 모델: 볼륨 마운트로 공유, 재다운로드 없음
- 정리 시 컨테이너/이미지 삭제하면 흔적 없음

---

## 참고: 나중에 Mac Studio 추가 시 분산 구성

Frontend/Router는 순수 Python(GPU 불필요)이므로 Mac에서 실행 가능:

```
Mac Studio                          DGX Spark
├── Frontend (python)    ── TCP →   ├── vLLM Worker (GPU)
├── Router   (python)               ├── NATS Server
└── env vars:                       └── etcd Server
    ETCD_ENDPOINTS=DGX_IP:2379
    NATS_SERVER=nats://DGX_IP:4222
```

핵심 파일: `components/src/dynamo/frontend/main.py`, `components/src/dynamo/router/__main__.py`

---

## 핵심 파일
- `container/build.sh` — ARM64 + CUDA 13 빌드 로직
- `container/run.sh` — 컨테이너 실행 (GPU, HF 캐시, 볼륨)
- `deploy/docker-compose.yml` — NATS + etcd 인프라
- `container/Dockerfile.vllm` — vLLM Dockerfile
