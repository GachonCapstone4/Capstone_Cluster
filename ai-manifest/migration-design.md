# AI 추론 서버 클러스터 마이그레이션 설계

> 작성일: 2026-09-28 / 방향 확정: 2026-09-30 / LLM 변경: 2026-10-01
> 대상 레포: https://github.com/GachonCapstone4/AI (기준 커밋 `19edfd2`)
> 상태: **클러스터 적용 완료 (2026-10-01)**. 결과는 7장 참고. AWS 서버 정리(5장 5·7단계)는 남아 있음

## 요약

**추론 서버를 클러스터(`maily`)로 옮기고, LLM은 클러스터 안의 Ollama(`ai-infra`)를 쓴다.** SageMaker 학습·재학습 MLOps와 S3 모델 저장소는 지금처럼 유지한다.
**AI 레포는 수정하지 않는다.** 필요한 차이는 환경변수(ConfigMap/Secret)와 클러스터 레포의 amd64 빌드 파일(빌드 시 패치 포함)로 흡수한다.

| 구분 | 결정 |
|---|---|
| 옮기는 것 | 추론 서버 (FastAPI + classify/deployment 컨슈머) |
| LLM | 학교 LLM `cellm.gachon.ac.kr` → **클러스터 Ollama** `ollama-service.ai-infra:11434` (`qwen2.5:3b-instruct`) |
| 유지하는 것 | SageMaker 학습·재학습, S3(`models/`, `latest.json`) |
| AI 레포 변경 | 없음 (LLM 타임아웃은 `Dockerfile.amd64`에서 빌드 시 패치) |
| 이미지 | 클러스터 레포에서 amd64로 별도 빌드 (`ai-manifest/build/`) |

---

## 1. 현재 구조와 이전 후 구조

| 구성요소 | 현재 | 이전 후 |
|---|---|---|
| 추론 서버 | AWS arm64(Graviton), 포트 8080 | **클러스터 `maily/ai-inference`**, 포트 8080 |
| 분류 컨슈머 | `q.2ai.classify` → `q.2app.classify` (서버 내 스레드) | 동일 |
| 배포 컨슈머 | `q.2ai.deployment` → preload → validate → switch | 동일 |
| 모델 | SBERT(MiniLM-L12) + LogisticRegression, S3 `models/{ver}/` | 동일 (S3에서 받아 emptyDir 캐시) |
| LLM (요약·일정 추출) | 학교 GPU 서버 (Qwen3.5-35B) | **클러스터 Ollama `qwen2.5:3b-instruct` (CPU 추론, RAG와 공유)** |
| 학습 | SageMaker + `Dockerfile.training` (backend/admin이 호출) | **동일** |
| 모니터링 | Prometheus → VPN `172.16.2.10:8080` | pod SD job `ai-inference` |

RabbitMQ와 Ollama는 이미 클러스터 안에 있다. 따라서 추론 파드에 필요한 외부 권한은 **S3 읽기** 하나뿐이다. SageMaker 권한과 학교 LLM 접속은 필요 없다.

### LLM 연결 방식 (코드 확인 결과)

- `api/services/llm_client.py`는 OpenAI 호환 `POST {base_url}/chat/completions`를 `requests`로 직접 호출한다. Ollama `/v1`이 이 형식을 지원하므로 **환경변수만으로 연결된다.**
- `LLM_PROVIDER=openai` + `OPENAI_BASE_URL`/`OPENAI_MODEL`을 쓴다. `school`도 동작하지만, 이름 때문에 학교 LLM으로 오해할 수 있어서 쓰지 않는다.
- 시작할 때 `OPENAI_API_KEY`가 비어 있으면 실패한다. Ollama는 인증을 하지 않으므로 Secret에 더미 값(`ollama`)을 넣는다.
- LLM은 classify 흐름의 `summarize_email()`(요약과 일정 추출)에서만 호출한다. 이때 `max_tokens=400`, `temperature=0.1`은 코드에 고정되어 있다.

### MLOps 흐름 (변경 없음)

```
admin/backend ──► SageMaker 학습 Job ──► S3 models/{ver}/ 업로드 + latest.json 갱신
                                     └──► q.2app.training (COMPLETED)
admin ──► x.app2ai.direct ─► q.2ai.deployment ─► ai-inference 파드
                                                 preload(S3) → validate → switch
classify ─► ai-inference ─► (요약) ollama-service.ai-infra:11434/v1 ─► q.2app.classify
```

학습 쪽은 추론 서버가 AWS에 있든 클러스터에 있든 관계없이 동작한다. 모든 연결이 S3와 RabbitMQ를 거치기 때문이다.

---

## 2. 코드 수정 없이 해결한 방법

| # | 문제 | 해결 (AI 레포 무변경) |
|---|---|---|
| 1 | 원본 Dockerfile이 `FROM --platform=linux/arm64`로 하드코딩되어 있고 노드는 전부 amd64 | `ai-manifest/build/Dockerfile.amd64`로 AI 레포 소스를 amd64로 별도 빌드 |
| 2 | amd64에서 `torch==2.3.0`을 받으면 CUDA 휠이 설치되어 이미지가 수 GB가 됨 | 위 Dockerfile에서 torch를 CPU 인덱스로 먼저 설치 (`2.3.0+cpu`가 `==2.3.0`을 충족) |
| 3 | `CMD`가 포트 8080으로 고정되어 `API_PORT`가 무시됨 | 매니페스트의 containerPort, 프로브, Service를 전부 8080으로 맞춤 |
| 4 | LLM provider의 API 키와 모델이 비어 있으면 시작 실패 | ConfigMap에 `OPENAI_BASE_URL`과 `OPENAI_MODEL`, Secret에 더미 `OPENAI_API_KEY` |
| 5 | **`llm_client.py`의 읽기 타임아웃이 60초로 하드코딩되어 있음** (`timeout=(5, 60)`) | `Dockerfile.amd64`에서 빌드할 때 `sed`로 패치 → `LLM_TIMEOUT_SECONDS`(기본 180)를 읽음. 패턴이 없으면 빌드 실패 (5-A 참고) |
| 6 | 재시작하면 모델 버전이 돌아감 (switch는 메모리에서만 일어남) | 3-B의 운영 규칙으로 처리 |
| 7 | RabbitMQ 토폴로지 없음 (NFS 마이그레이션 때 사라짐) | 컷오버 0단계에서 Terraform 재적용 |

### 2-A. 타임아웃 패치가 필요한 이유 (2026-10-01 실측)

maily 네임스페이스에서 `ollama-service.ai-infra`로, 실제 summarize 프롬프트 형식으로 측정했다.

| 케이스 | 소요 시간 |
|---|---|
| 짧은 메일(prompt 386 토큰, 출력 117 토큰), 모델 미적재(콜드) | **78.5s** |
| 같은 메일, 모델 적재 상태 | 35.6~36.9s |
| 약 3,000자 메일(prompt 1,842 토큰, 출력 212 토큰) | **271s** |

- 처리 속도는 prompt 약 9.5 tok/s, 생성 약 2.7 tok/s다. GPU 없이 CPU로만 돌고 Ollama limit은 cpu 3이다.
- 60초를 넘기면 `LLMTransientError`가 난다. `classify_service`는 이 오류를 잡지 않는다(`LLMPermanentError`만 "요약 생성 실패"로 대체). 그래서 메시지가 retry 큐로 가서 최대 3번 재시도되고, 그래도 안 되면 DLQ로 간다. **분류 결과가 아예 나가지 않고**, 재시도 요청이 Ollama 부하를 더 늘린다.
- 180초는 RAG의 `LLM_API_TIMEOUT_SECONDS: 180`과 같은 값이다. `LLM_MAX_INPUT_CHARS=3000`으로 입력을 제한하면 대부분 이 안에 끝난다.

---

## 3. 핵심 설계 결정

### A. 이미지 빌드: 클러스터 레포에서 amd64로 따로 빌드
- AI 레포 CI는 지금처럼 arm64 이미지(`capstoneai:latest`, `:<sha>`)를 계속 만든다. AWS 롤백용으로 그대로 쓸 수 있다.
- 클러스터용 이미지는 태그를 `suhannugul/capstoneai:amd64-<AI커밋sha7>`로 구분한다. `latest`는 쓰지 않는다.
- 빌드 방법
  - **GitHub Actions:** `.github/workflows/ai-image-amd64.yaml`을 `workflow_dispatch`로 수동 실행하고 `ai_ref`를 입력한다. 레포 시크릿 `DOCKERHUB_USERNAME`과 `DOCKERHUB_TOKEN`이 필요하다.
  - **로컬:**
    ```
    docker buildx build --platform linux/amd64 \
      -f ai-manifest/build/Dockerfile.amd64 \
      -t suhannugul/capstoneai:amd64-<sha7> --push <AI레포경로>
    ```
- 트레이드오프: AI 레포에 push해도 자동으로 빌드되지 않는다. 추론 코드를 바꾸면 워크플로를 수동으로 실행하고, `ai-pod.yaml`의 이미지 태그를 올려야 한다.
- 원본 Dockerfile이 바뀌면(의존성, COPY 대상 등) `Dockerfile.amd64`도 같이 맞춰야 한다.
- **빌드 패치:** `llm_client.py`의 `timeout=(5, 60),`을 `timeout=(5, float(os.getenv("LLM_TIMEOUT_SECONDS", "180"))),`로 바꾼다. 원본에서 이 줄이 바뀌면 `grep`에서 걸려 **빌드가 실패**한다. 그때 패치를 다시 맞춘다. AI 레포가 타임아웃을 환경변수로 받도록 바뀌면 이 패치는 삭제한다.

### B. 모델 버전: `ACTIVE_MODEL_VERSION`은 고정값으로 두고 운영 규칙으로 관리
- `ACTIVE_MODEL_VERSION`을 비워 두면 시작할 때 `latest.json`을 읽는다. 그런데 학습 컨테이너는 **학습이 끝나자마자** `latest.json`을 갱신한다. 그래서 재시작하면 validate를 거치지 않은 모델이 올라갈 위험이 있다. **비워 두면 안 된다.**
- 운영 규칙: `/deployment` switch에 성공하면 `ai-configmap.yaml`의 `ACTIVE_MODEL_VERSION`을 새 버전으로 올려서 커밋하고 적용(ArgoCD 동기화)한다.
- ConfigMap을 바꾸는 것만으로는 파드가 재시작되지 않는다. 이미 메모리에서 switch된 상태이므로 재시작할 필요도 없다.

### C. 캐시: emptyDir
- 모델 결과물은 수백 MB이고 원본은 S3에 있으므로, 새로 뜰 때마다 다시 받는다.
- 노드에 고정할 필요도 RWO 제약도 없어져 **RollingUpdate**(maxSurge 1, maxUnavailable 0)를 쓸 수 있다. hostPath PV(`ai-volume.yaml`)는 삭제했다.

### D. 레플리카 1개
- `q.2ai.deployment` 메시지는 파드 하나만 받아 간다. 레플리카를 늘리면 모델 버전이 섞인다.
- 늘리려면 코드를 바꿔야 하므로(배포 이벤트 fanout 등) 이번 범위에서 제외한다.
- 롤링 업데이트 중에는 잠깐 파드가 2개가 된다. 이 시간에는 deployment 이벤트를 보내지 않는다.

### E. 리소스
- requests `cpu 500m / memory 1.5Gi`, limits `cpu 2 / memory 3Gi`로 한다. staging 모델을 같이 올리면 메모리가 약 2배가 된다.
- `OMP_NUM_THREADS`와 `MKL_NUM_THREADS`는 2로 둔다.
- 워커는 각각 **6코어, 약 9.6Gi**다(2026-10-01 확인). worker-1에는 Ollama(limit cpu 3 / 6Gi)가 고정되어 있고, worker-1의 memory limit 합계는 이미 78%다. ai-inference(limit 3Gi)가 worker-1에 올라가면 limit 합계가 100%를 넘는다. 배치 노드는 스케줄러에 맡기되, worker-1에 올라가면 메모리 압박을 지켜본다.

### F. 모니터링
- 기존 `spring-boot-actuator` job은 `prometheus.io/scrape` annotation이 붙은 모든 파드를 수집한다. 중복 수집을 피하려고 AI 파드에는 **annotation을 붙이지 않는다.**
- 대신 전용 job `ai-inference`를 둔다. `maily` 네임스페이스에서 `app=ai-inference` 라벨과 포트 8080으로 수집한다.
- `aws-ai-server` job은 컷오버 동안 비교용으로 남기고, AWS 서버를 종료한 뒤 삭제한다.
- Grafana `ai-inference-monitoring` 대시보드 쿼리가 `job="aws-ai-server"`로 되어 있다면 `job=~"aws-ai-server|ai-inference"`로 바꿔야 한다.

### G. LLM: 클러스터 Ollama
- 엔드포인트는 `http://ollama-service.ai-infra.svc.cluster.local:11434/v1`이고, 모델은 `qwen2.5:3b-instruct`다. RAG와 같은 모델이라 메모리가 추가로 들지 않는다.
- Ollama 설정(`ollama-manifest/03-ollama-deployment.yaml`)
  - `OLLAMA_KEEP_ALIVE=-1`: 모델을 계속 메모리에 둔다. 기본값(5분)이면 언로드된 뒤 첫 요청에서 콜드 스타트가 약 78초 걸린다.
  - `OLLAMA_NUM_PARALLEL=1`: CPU 추론에서는 병렬 처리가 경합만 늘린다. ai와 RAG 요청은 순서대로 처리한다.
- `LLM_MAX_INPUT_CHARS=3000`: 원래 12000이었다. 이 값을 그대로 두면 처리에 수 분이 걸리고 Ollama context(4096)도 넘는다.
- **알려진 한계**
  - 처리량: 1건에 약 40초~3분이 걸리고 Ollama는 1개다. RAG와 같이 쓰기 때문에 메일이 몰리면 `q.2ai.classify`가 쌓인다. 필요하면 Ollama cpu limit을 올린다(노드가 6코어라 한계가 있음).
  - 품질: 3B 모델이라 추출 정확도가 학교 LLM(35B)보다 낮다. 실측에서 `"다음주 화요일"`이 `"화요일"`로 잘렸고, location에 템플릿 문구 `"(없으면 null)"`이 섞여 나왔다. 컷오버 4단계에서 확인한다.
  - 입력 잘라내기: `truncate_input`은 프롬프트 전체 기준으로 자른다. 그래서 긴 메일은 끝부분의 `[출력 형식]`까지 잘려 JSON 파싱에 실패하고, 응답 원문이 그대로 summary로 들어간다(오류는 나지 않음).

---

## 4. 매니페스트 구성

| 파일 | 내용 |
|---|---|
| `ai-configmap.yaml` | `API_PORT=8080`, `ACTIVE_MODEL_VERSION` 고정, Ollama LLM(`LLM_PROVIDER=openai`, `OPENAI_*`, `LLM_TIMEOUT_SECONDS`), CPU 튜닝 |
| `secret/00-ai-secret.yaml` (git 제외) | `RABBITMQ_URL`, `AWS_ACCESS_KEY_ID/SECRET` (S3 읽기 권한만), `OPENAI_API_KEY=ollama` (더미) |
| `ai-pod.yaml` | Deployment ×1 (RollingUpdate, emptyDir, 프로브 8080) + ClusterIP Service 8080 |
| `build/Dockerfile.amd64` | amd64 + torch CPU 빌드 + LLM 타임아웃 패치 |
| `../.github/workflows/ai-image-amd64.yaml` | 수동 실행하는 이미지 빌드 워크플로 |
| `../monitoring-manifest/03-prom-config.yaml` | `ai-inference` job 추가 |
| `../ollama-manifest/03-ollama-deployment.yaml` | `OLLAMA_KEEP_ALIVE=-1`, `OLLAMA_NUM_PARALLEL=1` 추가 |

- Ingress는 두지 않는다. backend와 admin은 RabbitMQ로만 통신하고, `/deployment/*` HTTP도 클러스터 안에서만 쓴다.
- Docker Hub 이미지가 public이므로 `imagePullSecrets`는 두지 않는다.
- 현재 클러스터의 `ai-secret`에는 `SCHOOL_LLM_API_KEY`가 들어 있다. `OPENAI_API_KEY`를 추가하고 `SCHOOL_LLM_API_KEY`는 삭제한다.

---

## 5. 컷오버 절차 (무중단)

RabbitMQ의 competing consumer 구조를 이용하면 AWS 서버와 클러스터 파드를 **동시에 돌려도 안전하다**. 두 서버가 메시지를 나눠 처리하므로 그 자체가 카나리 역할을 한다.

0. **사전 준비**
   - Terraform으로 RabbitMQ 토폴로지를 복구한다. 필요한 것: `x.app2ai.direct`, `x.ai2app.direct`, `x.retry.direct`, `x.sse.fanout`, `q.2ai.*`, `q.2app.*`, `q.dlx.failed`, 그리고 DLX 인자.
   - 클러스터에서 S3로 나가는 egress가 되는지 확인한다.
   - Ollama를 확인한다. 모델 목록(`kubectl -n ai-infra exec deploy/ollama -c ollama -- ollama list`)에 `qwen2.5:3b-instruct`가 있어야 한다. `OLLAMA_KEEP_ALIVE`, `OLLAMA_NUM_PARALLEL`을 반영해 재배포한 뒤, `ollama ps`의 UNTIL이 `Forever`인지 확인한다.
   - `ai-secret`의 `RABBITMQ_URL` 호스트가 실제 Service인 `rabbitmq-headless.rabbitmq.svc.cluster.local`인지 확인한다. rabbitmq 네임스페이스에 `rabbitmq`라는 이름의 Service는 없다.
   - `ai-secret`에 `OPENAI_API_KEY`가 있는지 확인한다.
   - ConfigMap의 `ACTIVE_MODEL_VERSION`을 AWS 서버에서 쓰는 값과 같게 맞춘다.
1. amd64 이미지를 빌드하고 푸시한다. 빌드 로그에서 타임아웃 패치 단계가 통과했는지 확인하고, `ai-pod.yaml`의 태그와 일치하는지 확인한다.
2. Secret → ConfigMap → `ai-pod.yaml` 순서로 적용하고, Prometheus ConfigMap도 적용한 뒤 reload한다.
3. 로그에서 `model_manager_ready`와 `consumer_started`를 확인하고, `/health`와 `/metrics`도 확인한다. `llm_outgoing_request` 로그의 url이 `ollama-service.ai-infra`인지도 확인한다.
4. 이제 두 서버가 메시지를 나눠 처리한다. Grafana에서 `ai_classify_latency_seconds`와 `ai_classify_errors_total`을 AWS 쪽과 비교한다.
   - 클러스터 쪽은 LLM이 느리므로 latency가 수십 초 늘어나는 것이 정상이다. 대신 retry나 DLQ(`q.dlx.failed`)가 늘어나는지, `q.2ai.classify`가 쌓이는지 확인한다.
   - 같은 종류의 메일에 대해 양쪽 summary와 schedule 품질을 샘플로 비교한다.
   - 이 기간에는 **deployment 이벤트(모델 교체)를 보내지 않는다.** 한쪽 서버만 교체된다.
5. 문제가 없으면 **AWS 서버를 중지**한다. 인스턴스는 롤백용으로 며칠 남겨 둔다.
6. `/deployment` preload → validate → switch를 E2E로 한 번 돌려 본다(`scripts/check_rabbitmq_e2e.py`). 성공하면 3-B 규칙대로 ConfigMap 버전을 올린다.
7. 안정화되면 `aws-ai-server` scrape job을 삭제한다.
8. **롤백:** AWS 서버를 다시 켜고 `kubectl -n maily scale deploy/ai-inference --replicas=0`을 실행한다. 큐 구조가 같으므로 추가 작업은 없다. AWS 서버는 계속 학교 LLM을 쓴다.

---

## 7. 적용 결과 (2026-10-01)

### 계획과 달랐던 점

| 항목 | 계획 | 실제 | 조치 |
|---|---|---|---|
| `ai-secret` `RABBITMQ_URL` | 맞는지 확인만 | 호스트가 `rabbitmq.rabbitmq.svc`였음 (이 Service는 없음) | `rabbitmq-headless.rabbitmq.svc.cluster.local`로 수정 (계정 유지) |
| `ACTIVE_MODEL_VERSION` | `2026-04-14-001` | 이 버전이 S3에 없음 | `training-final-004`로 변경 (latest.json과 같고 S3에 있음) |
| RabbitMQ 토폴로지 | 0단계에서 Terraform 재적용 | 이미 복구되어 있음 (`x.app2ai.direct`, `x.ai2app.direct`, retry 바인딩 확인) | 생략 |
| 카나리 (AWS와 동시 처리) | 두 서버가 메시지를 나눠 처리 | 적용 전 `q.2ai.classify` 컨슈머가 0 (AWS 서버가 메시지를 받지 않음) | 불가. 클러스터 파드가 전체 트래픽을 받음 |
| Prometheus | 로컬 `03-prom-config.yaml`을 적용한 뒤 reload | 로컬 파일에 커밋되지 않은 다른 변경이 있음 (`node2-mc-host` 삭제, `backend` job 추가) | 클러스터 설정에 `ai-inference` job만 추가. `--web.enable-lifecycle`이 없어 SIGHUP으로 reload |
| 이미지 빌드 | GitHub Actions 또는 로컬 | 로컬 | `git archive 19edfd2`로 소스만 꺼내 빌드 (AI 레포 작업 트리는 건드리지 않음) |
| `ai-secret` | `OPENAI_API_KEY` 추가 | 추가함 (`ollama`) | `SCHOOL_LLM_API_KEY`는 쓰지 않지만 남아 있음 |

### 검증

- 이미지 `suhannugul/capstoneai:amd64-19edfd2` (digest `sha256:81780820…`): amd64, torch `2.3.0+cpu`, `llm_client.py:158` 타임아웃 패치 적용 확인.
- Ollama: `OLLAMA_KEEP_ALIVE=-1`, `OLLAMA_NUM_PARALLEL=1` 반영. `ollama ps`에서 qwen2.5:3b-instruct가 `UNTIL Forever`.
- ai-inference: worker-2에서 Running. `model_manager_ready(training-final-004)`, `/health` ok, `q.2ai.classify`와 `q.2ai.deployment` 컨슈머 각 1.
- E2E (outbox_id `999999001`, 결과 메시지는 확인 후 큐에서 삭제)
  - 요청부터 `q.2app.classify` 도착까지 **약 82초**. 원래 60초 타임아웃이었다면 실패했을 시간이다.
  - LLM 호출이 `ollama-service.ai-infra`로 가는 것을 확인했고, retry나 DLQ로 빠지지 않았다.
  - 결과: `Sales / 미팅 일정 조율`, summary 정상, schedule `2026-10-06 14:00 본사 3층 회의실` (다음주 화요일로 정확히 해석됨).
- Prometheus `ai-inference` target `up`.

### 남은 확인 사항

- **`q.2app.classify` 컨슈머 0**: backend가 분류 결과 큐를 받고 있지 않다. 결과가 큐에 쌓이므로 backend 쪽 컨슈머 상태를 확인해야 한다.
- 로컬 `monitoring-manifest/03-prom-config.yaml`이 클러스터 설정과 다르다. 이 파일을 적용하면 `node2-mc-host`가 삭제되니, 적용 전에 정리한다.
- `ai-manifest`와 `ollama-manifest`는 ArgoCD 앱으로 등록되어 있지 않다. 변경은 `kubectl apply`로 직접 적용한다.

---

## 6. 범위 밖 (나중에 필요할 때)

| 대상 | 이유 |
|---|---|
| 학습을 K8s Job으로 이전 | SageMaker 유지로 결정 |
| S3를 MinIO로 교체 | SageMaker가 S3를 전제로 동작 |
| 스케일아웃과 HPA | 배포 이벤트 브로드캐스트 구조로 코드를 바꿔야 함 |
| 모델 버전 자동 유지 (`active.json`) | 코드 수정 필요, 지금은 운영 규칙으로 대체 |
| AI 레포에 타임아웃 환경변수 반영 | 반영되면 `Dockerfile.amd64`의 빌드 패치 삭제 |
| LLM 처리량/품질 개선 (GPU 노드, 더 큰 모델, `LLMTransientError` fallback 처리) | 하드웨어 추가 또는 코드 수정이 필요 |
