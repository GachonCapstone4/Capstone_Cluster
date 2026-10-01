# MariaDB 1020 오류: classify 결과 저장 시 `calendar_events` INSERT 실패

> 발견일: 2026-10-01 (AI 추론 서버 클러스터 이전 후 E2E 테스트 중)
> 대상: backend `MailServiceImpl.markFinished()` / MariaDB 12.3.3 (`database/mariadb-0`)
> 상태: **원인 분석 완료. 임시 조치(DB 설정 OFF) 적용 완료 (2026-10-01 17:55). 코드 수정은 backend 담당 판단 필요**

---

## 1. 증상

DB 주입 방식으로 E2E 테스트를 했다(`emails` 1건과 `outbox` READY 1건을 넣음). classify 결과를 처리하던 backend의 **1차 시도가 실패**했고, 30초 뒤 **재시도에서 성공**했다.

```
17:23:54.678 WARN  SQL Error: 1020, SQLState: HY000
17:23:54.679 ERROR Record has changed since last read in table 'calendar_events'; try restarting transaction
17:23:54.700 ERROR [MailService] CalendarEvent 생성 실패 (파싱 오류) — emailId=1 ...
17:23:54.721 INFO  [MailService] classify 완료 — outboxId=1, emailId=1
17:23:54.755 ERROR [MailConsumer] 처리 실패 → nack — outboxId=1,
                   error=Transaction silently rolled back because it has been marked as rollback-only
...
17:24:24.862 INFO  [MailService] classify 완료 — outboxId=1, emailId=1   ← 재시도 성공
```

- 로그에는 "파싱 오류"라고 찍히지만, 실제 원인은 DB 오류 1020이다. `createPendingCalendarEventIfAbsent`의 catch가 모든 예외를 이 문구로 기록한다.
- catch로 예외를 잡았지만 Spring이 이미 트랜잭션을 **rollback-only**로 표시한 뒤다. 그래서 `markFinished` 전체가 롤백된다(outbox FINISH, 분석 결과, 알림 모두 포함).
- 3회 연속 실패하면 outbox가 `FAILED`로 끝나고 메시지는 `q.dlx.failed`로 간다.

## 2. 배경: `innodb_snapshot_isolation`

| 항목 | 값 |
|---|---|
| MariaDB 버전 | 12.3.3 |
| `innodb_snapshot_isolation` | **1 (ON)**. 11.6부터 기본값이 ON |
| `transaction_isolation` | REPEATABLE-READ |

- REPEATABLE-READ 트랜잭션은 첫 consistent read 시점의 read view(스냅샷)를 기준으로 데이터를 읽는다.
- 잠금 읽기(INSERT, UPDATE, `FOR UPDATE`, **FK 부모 행 검사**)는 최신 커밋 버전을 본다.
  - **OFF (11.5 이전 동작)**: 최신 버전으로 그대로 진행한다.
  - **ON**: 잠금 읽기로 건드린 행이 **내 read view 이후 다른 트랜잭션이 커밋한 버전**이면 `ER_CHECKREAD (1020)`을 낸다.
- `calendar_events`에 INSERT하면 FK(`fk_calendar_user` → `users`, `fk_calendar_email` → `emails`) 검사로 부모 행을 잠금 읽기한다. **부모 행에서 충돌해도 에러 메시지에는 INSERT 대상 테이블(`calendar_events`)이 찍힌다.**

### 로컬 재현 (mariadb:12.3, 같은 설정)

트랜잭션 A가 `BEGIN → SELECT(read view 생성) → 2초 대기 → INSERT calendar_events(user_id=1, email_id=1)`을 실행하는 동안, 트랜잭션 B가 아래 작업을 하고 커밋했다.

| B가 한 일 | A의 INSERT |
|---|---|
| 없음 | 성공 |
| `users(1)`을 참조하는 자식 행 INSERT (rag_jobs 형태) | 성공 |
| `users(1)` 잠금 읽기 (`LOCK IN SHARE MODE`) | 성공 |
| **`users(1)` UPDATE** | **ERROR 1020 ... in table 'calendar_events'** |
| **`emails(1)` UPDATE** | **ERROR 1020 ... in table 'calendar_events'** |

→ 다른 트랜잭션이 **부모 행을 UPDATE하고 커밋**하면 운영과 똑같은 오류가 난다.

## 3. 원인: 다른 트랜잭션이 `users(1)`을 UPDATE함

`emails(1)`의 `updated_at`은 주입 시각 그대로라서, 충돌한 행은 `users(1)`이다.

### 1차 시도 타임라인

```
17:23:54.2   T1 시작 — MailServiceImpl.markFinished (@Transactional)
             outbox 조회 → T1의 read view 생성
             email_analysis_results INSERT
17:23:54.57  ragIntegrationService.requestTemplateMatch()
             └ RagPublisher.publishTemplateMatch()
               └ RagJobService.createTemplateMatchJob  @Transactional(REQUIRES_NEW) → 별도 트랜잭션 T2
                   getOrCreate: job 없음 → resolveUser() 로 User 엔티티 로드
                   rag_jobs INSERT, 커밋 시 User가 dirty로 판단 → UPDATE users(1) → 커밋   ★
               └ RabbitMQ 발행 (트랜잭션과 무관하게 즉시 나감)
17:23:54.67  T1: calendar_events INSERT → FK 검사로 users(1) 잠금 읽기
             → 최신 버전은 T1 read view 이후 T2가 커밋한 것 → 1020
             → T1 rollback-only → 커밋 시 UnexpectedRollbackException → nack → retry 큐
```

### 재시도가 성공한 이유

1차 시도의 T2가 `rag_jobs.template-match-1`을 이미 커밋해 두었다. 재시도 때는 `findById`로 기존 job을 찾았으므로 `resolveUser()`가 호출되지 않았고, `users` UPDATE도 없었다.

### User를 로드하기만 해도 UPDATE가 나가는 이유 (유력)

- 코드에서 User를 수정하는 부분은 없다. 그런데 `users.updated_at = 17:24:27.366`으로, RAG 결과 처리(`rag_jobs.completed_at = 17:24:27.384`)와 같은 시각이다. **RAG job 경로에서 User를 로드하면 UPDATE가 나간다**는 증거다.
- `User.displayConfigs`는 `@Convert(DisplayConfigConverter)`로 매핑된 `UserDisplayConfig` 객체다. 이 클래스와 내부 `Widget`에는 **`equals/hashCode`가 없다**(`@Getter/@Setter`만 있음).
- Hibernate는 converter가 붙은 속성의 dirty 여부를 로드 시 스냅샷 복사본과 현재 값의 `equals`로 비교한다. `equals`가 없으면 항상 다르다고 판단한다. 그래서 **User를 로드한 트랜잭션은 flush할 때마다 `UPDATE users`를 실행한다** (`@UpdateTimestamp`로 `updated_at`도 갱신됨).

### 확인 수준

| 내용 | 근거 |
|---|---|
| 부모 행 UPDATE가 1020을 일으키고, 메시지에 `calendar_events`가 찍힘 | ✅ 같은 버전에서 재현 |
| 충돌한 행은 `users(1)` | ✅ `emails.updated_at`이 바뀌지 않음 |
| RAG job 경로가 User를 로드하면 `users`가 UPDATE됨 | ✅ `users.updated_at`과 `rag_jobs.completed_at`이 일치 |
| 17:23:54의 T2가 `users`를 UPDATE함 | ⚠️ 추정. 이후 값이 덮어써졌고 binlog는 backend 계정 권한(`BINLOG MONITOR`)이 없어 확인 못 함 |
| UPDATE 원인이 `UserDisplayConfig`의 `equals` 누락 | ⚠️ 코드상 가장 유력. SQL 로그로 직접 관찰하지는 못함 |

## 4. 영향

- 일정이 감지된 메일이면서 그 사용자의 RAG job이 새로 생기는 경우 1차 시도가 실패한다. 재시도로 대개 복구되지만 30초가 늦어진다.
- 롤백과 관계없이 **RAG template match 요청이 중복 발행**된다.
- User를 로드하는 다른 트랜잭션(설정 조회, 알림 등)과 겹치면 다른 경로에서도 같은 1020이 날 수 있다. 사용자가 늘수록 빈도가 늘어난다.

## 5. 조치

### 5-1. 임시 조치: `innodb_snapshot_isolation=OFF` — ✅ 적용 완료 (2026-10-01 17:54~17:55)

`SET GLOBAL`로 런타임에 바꾸는 방식은 자동 승인이 거부되었다. 그래서 **설정 파일에 넣고 롤링 재시작**하는 방식으로 영구 적용했다.

**변경 (`mariadb-manifest/`, 클러스터에서 추출한 매니페스트 기준)**
- `02-mariadb-configmap.yaml`: `primary.cnf`와 `replica.cnf`의 `[mysqld]` 섹션에 `innodb_snapshot_isolation=OFF` 추가.
- `04-mariadb-statefulset.yaml`: 이미지를 `mariadb:latest`에서 `mariadb@sha256:ab1c3dd3…`(실행 중이던 12.3.3 digest)로 고정. initContainer와 메인 컨테이너 둘 다.
  - **이유:** `imagePullPolicy: Always`이고, 그 사이 Docker Hub의 `latest`가 **13.0.2**로 바뀌었다. 고정하지 않고 재시작하면 12.3.3 데이터 디렉터리 위에서 13.0.2가 떠서, 되돌릴 수 없는 메이저 업그레이드가 된다. 이번에 12.x가 올라와 snapshot isolation이 기본으로 켜진 것도 `latest` 사용 때문으로 보인다.

**절차**
1. 같은 digest 이미지에 수정한 cnf를 넣어 로컬에서 기동 검증: 12.3.3, `innodb_snapshot_isolation=0`, ERROR 없음.
2. `kubectl diff`로 변경 범위가 이미지 2줄과 cnf 설정 2줄뿐인지 확인.
3. ConfigMap을 적용한 뒤 StatefulSet을 적용. template이 바뀌므로 롤링 업데이트가 자동으로 진행된다(mariadb-1 다음 mariadb-0, 약 1분 30초).

**검증 결과**

| 확인 항목 | mariadb-0 | mariadb-1 |
|---|---|---|
| 버전 | 12.3.3 | 12.3.3 |
| `innodb_snapshot_isolation` | OFF | OFF |
| `read_only` | OFF | NO_LOCK_NO_ADMIN |
| `gtid_current_pos` | 0-1-427 | 0-1-427 (복제 일치) |

- 복제: primary가 재시작되는 동안 연결 거부(2003)가 났다가 17:55:45에 다시 연결됐다.
- backend: primary 재시작 구간(17:55:18)에 Hikari 커넥션 타임아웃 1건이 있었고, 17:55:40 이후 오류는 0건이다. admin은 0건.
- backend와 admin은 재시작하지 않았다. DB가 재시작되면서 기존 커넥션이 끊겼고, 새 커넥션은 OFF 설정을 받는다.

**주의**
- 이 매니페스트 폴더는 ArgoCD가 관리하지 않는다. 변경은 `kubectl apply`로 직접 적용한다.
- 근본 수정(5-2)이 끝나면 OFF를 유지할지, 다시 ON으로 돌릴지 검토한다.
- 부작용: DB 전체(admin 등 다른 앱 포함)가 11.5 이전 REPEATABLE-READ 동작으로 돌아간다. lost update를 막아 주던 장치가 없어지므로, 근본 수정(5-2) 후에는 다시 ON으로 되돌리는 것을 검토한다.

### 5-2. 근본 수정 (backend 레포, 담당 확인 필요)

1. **`UserDisplayConfig`와 `UserDisplayConfig.Widget`에 `@EqualsAndHashCode` 추가**. 불필요한 `UPDATE users`가 사라진다. 수정 범위가 가장 작고 효과가 크다.
2. **RAG template match 발행을 커밋 이후로 이동**. `requestTemplateMatch`를 `@TransactionalEventListener(phase = AFTER_COMMIT)`으로 옮긴다. 트랜잭션 도중 REQUIRES_NEW 트랜잭션이 끼어드는 구조가 사라지고, 롤백됐을 때 RAG 요청이 중복 발행되는 문제도 해결된다.
3. (선택) `createPendingCalendarEventIfAbsent`의 catch 로그 문구를 바로잡는다. 실제로는 DB 오류인데 "파싱 오류"로 찍힌다.

## 6. 관련 테스트 데이터

E2E 테스트 행은 아직 운영 DB에 남아 있다. 식별값은 `external_msg_id = 'e2e-test-20261001-001'`이고, 대상은 `emails`, `outbox`, `email_analysis_results`, `calendar_events`, `notifications` 2건, `rag_jobs`(`template-match-1`)다.
정리 SQL은 FK 순서대로 지우도록 별도로 준비해 두었다.
