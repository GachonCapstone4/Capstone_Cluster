# NFS 마이그레이션 진행상황

> 작성일: 2026-09-28
> 대상: ollama, chroma, prometheus, grafana, rabbitmq — 전부 local-path/hostPath → NFS(192.168.2.40) 정적 PV로 전환

## 배경

기존에는 `rancher.io/local-path` 동적 프로비저닝(ollama/chroma/rabbitmq) 또는 hostPath 정적 PV(grafana/prometheus)를 사용 중이었음.
데이터 유실 위험(노드 장애 시 로컬 디스크 데이터 손실) 때문에 NFS 서버(`192.168.2.40:/srv/nfs/share/*`)로 전부 이전.

**중요 결정**: 마이그레이션 시 기존 데이터는 백업하지 않고 삭제 후 새 볼륨으로 시작하기로 함 (사용자 명시적 지시).

## 매니페스트 변경 내역 (완료, 커밋 전)

| 파일 | 변경 내용 |
|---|---|
| `ollama-manifest/01-ollama-storageclass.yaml` → `01-ollama-pv.yaml` | StorageClass 삭제, NFS PV(`ollama-pv`, `/srv/nfs/share/ollama`, claimRef `ollama-data-pvc/ai-infra`)로 교체 |
| `ollama-manifest/02-ollama-pvc.yaml` | `storageClassName: ollama-storage` → `""` |
| `chroma-manifest/chroma-storageclass.yaml` → `chroma-pv.yaml` | NFS PV(`chroma-pv`, `/srv/nfs/share/chroma`, claimRef `chroma-data-chroma-0/database`) |
| `chroma-manifest/chroma-statefulset.yaml` | volumeClaimTemplate `storageClassName: chroma-storage` → `""` |
| `rabbitmq-manifest/rabbitmq-storageclass.yaml` → `rabbitmq-pv.yaml` | NFS PV(`rabbitmq-pv`, `/srv/nfs/share/rabbitmq`, claimRef `rabbitmq-data-rabbitmq-0/rabbitmq`) |
| `rabbitmq-manifest/rabbitmq-statefulset.yaml` | volumeClaimTemplate `storageClassName: rabbitmq-storage` → `""` |
| `monitoring-manifest/grafana/00-grafana-pv.yaml` | hostPath → nfs(`/srv/nfs/share/grafana`), claimRef 추가 |
| `monitoring-manifest/grafana/01-grafana-pvc.yaml` | `storageClassName: ""` 명시 추가 |
| `monitoring-manifest/04-prom-pv.yaml` | hostPath → nfs(`/srv/nfs/share/prometheus`) (이미 진행됨), `storageClassName: ""` 명시 추가 |
| `monitoring-manifest/05-prom-pvc.yaml` | `storageClassName: ""` 명시 추가 |
| `monitoring-manifest/06-prom-deploy.yaml` → `06-prom-statefulset.yaml` | **클러스터 드리프트 수정**: 실제 클러스터엔 Deployment가 아니라 StatefulSet(`prometheus-statefulset`)이 떠 있었음 → 매니페스트를 StatefulSet으로 정정 |

모든 PV/PVC/volumeClaimTemplate은 `storageClassName: ""`로 명시하여 정적 바인딩 강제 (default StorageClass 존재 여부와 무관하게 동작).

## 클러스터 적용 절차 (완료)

서비스별로 아래 순서로 실행함 (grafana는 사용자가 별도로 이미 완료):

1. `kubectl scale deployment/statefulset <name> -n <ns> --replicas=0`
2. 기존 PVC/PV `kubectl delete` (백업 없이 삭제 — 사용자 지시)
3. 노드에 남은 local-path 실제 디렉토리(`/opt/local-path-provisioner/pvc-*`)를 busybox 임시 Pod(hostPath + nodeSelector)로 `rm -rf` 하여 정리
   - ollama: k8s-worker-1, k8s-worker-2 (이전 테스트로 남은 고아 디렉토리 포함) 정리
   - chroma: k8s-worker-2 정리
   - rabbitmq: k8s-worker-1 정리 (고아 디렉토리 1개 포함)
   - prometheus: hostPath `/data/prometheus-data` (k8s-worker-2) 내용 정리
4. 새 PV(`kubectl apply`) → 새 PVC 적용
   - **주의**: StatefulSet의 `volumeClaimTemplates`는 in-place 수정이 Forbidden이라, chroma/rabbitmq는 StatefulSet 자체를 `kubectl delete` → `kubectl apply` 로 재생성해야 했음 (replicas=0 상태였어서 파드 영향 없음)
   - PV 이름이 기존과 동일한 경우(grafana-pv, prometheus-pv)는 `persistentVolumeSource`가 immutable이라 PV도 delete → apply 필요했음
5. `kubectl scale ... --replicas=1` 로 재기동
6. API 테스트로 정상 반영 확인

## 최종 상태 (2026-09-28 기준, 전부 Bound + 정상)

| 서비스 | PV | NFS path | 워크로드 상태 | API 확인 |
|---|---|---|---|---|
| grafana | grafana-pv | /srv/nfs/share/grafana | Deployment 1/1 | (사용자 별도 확인) |
| prometheus | prometheus-pv | /srv/nfs/share/prometheus | StatefulSet(prometheus-statefulset) 1/1 | `/-/healthy`, `/-/ready` OK |
| ollama | ollama-pv | /srv/nfs/share/ollama | Deployment 1/1 | `ollama list` → bge-m3, qwen2.5:3b-instruct 재다운로드 완료 |
| chroma | chroma-pv | /srv/nfs/share/chroma | StatefulSet(chroma) 1/1 | readiness probe(`/api/v1/heartbeat`) 통과 |
| rabbitmq | rabbitmq-pv | /srv/nfs/share/rabbitmq | StatefulSet(rabbitmq) 1/1 | `rabbitmq-diagnostics check_running` OK, `rabbitmqctl list_vhosts` → `/`만 존재 |

정리 완료: `chroma-storage` / `ollama-storage` / `rabbitmq-storage` local-path StorageClass 3개 삭제됨.

## ⚠️ 후속 작업 필요 (미완료)

### 1. RabbitMQ — Terraform 재적용 필요 (가장 시급)
- `/var/lib/rabbitmq` 볼륨을 백업 없이 새 것으로 교체했기 때문에 **Terraform(`cyrilgdn/rabbitmq` provider 등)으로 선언했던 모든 vhost(기본 `/` 제외)/큐/exchange/바인딩/유저/권한이 브로커에서 전부 사라짐**.
- rabbitmq-configmap.yaml에 `definitions.json` 자동 로드 설정이 없어서 재기동해도 자동 복구 안 됨.
- Terraform state 자체(로컬/remote backend)는 클러스터 밖에 있으므로 영향 없음 → 리소스 정의는 살아있음.
- **TODO**: terraform 폴더에서
  1. `terraform plan` 실행하여 drift 확인 (provider가 404 감지 시 재생성으로 뜨는지, 혹은 에러가 나는지 확인)
  2. plan에서 변화가 안 잡히면 `terraform apply -refresh=true` 혹은 개별 리소스 `terraform taint` 후 apply
  3. 재생성 후 연동 서비스(백엔드 등)가 정상적으로 큐에 publish/consume 가능한지 확인

### 2. Chroma — 벡터 데이터 재적재 필요
- 컬렉션이 전부 빈 상태로 새로 시작됨. 애플리케이션에서 임베딩을 재생성/재적재하는 로직이 있는지 확인 필요.

### 3. Ollama — 완료 (재다운로드 자동 처리됨), 추가 작업 없음

### 4. Grafana — 대시보드/데이터소스가 기본값으로 초기화됐을 가능성 있음 (사용자가 직접 진행했던 부분이라 별도 확인 권장)

## 참고 명령 (재확인용)

```bash
kubectl get pv
kubectl get pvc -A
kubectl get pods -n rabbitmq -o wide
kubectl exec rabbitmq-0 -n rabbitmq -- rabbitmqctl list_vhosts
kubectl exec rabbitmq-0 -n rabbitmq -- rabbitmqctl list_queues
kubectl exec rabbitmq-0 -n rabbitmq -- rabbitmqctl list_exchanges
```
