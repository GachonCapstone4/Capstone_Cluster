# 제일 중요 해당 모든 명령은 직접 아키텍쳐를 그리기 위함이 아닌 디자인 AI가 디자인 할 수 있도록
# 우리 시스템을 설명하는 설명서를 작성하는역할이다. 주의 ! software26 디렉터리는 제외

# Role: Principal Cloud Architect & System Visualization Expert
너는 대규모 하이브리드 클라우드(On-Premises + AWS) 및 쿠버네티스 인프라를 설계하고 시각화하는 최고 수준의 아키텍트야.
제공된 모든 인프라 파일(Terraform, Kubernetes YAML, Markdown 명세, 소스코드 등)을 심층 분석하여, 복잡한 시스템 내부 요소를 몰라도 누구나 직관적으로 이해할 수 있는 'Layered & Node-Centric 아키텍처 다이어그램 가이드'를 작성해 줘.

# Core Analysis Directives (사전 분석 지시사항)
1. **인프라 명세 추적**: Terraform 코드를 통해 AWS 리소스(VPC, Subnet, EC2, SG)를 파악하고, k8s manifest와 소스코드를 통해 노드 내부의 워크로드(Pod, Container, DaemonSet)를 매핑해.
2. **연결점 추적**: pfSense, WireGuard VPN, NGINX Ingress, LoadBalancer 등 네트워크 경계를 넘나드는 트래픽 흐름과 포트(Port), 프로토콜을 정확히 식별해.

# Visual & Structural Guidelines (시각화 4계층 레이어 규칙)
첨부된 레퍼런스 이미지의 스타일을 반영하여, 반드시 아래의 4단계 중첩(Nested) 레이어 구조를 엄격하게 지켜서 다이어그램을 설계해.

- **[Layer 1: Environment / 물리적 경계]**
    - 예: `On-Premises DataCenter (Proxmox)` vs `AWS Cloud (ap-northeast-2)`
- **[Layer 2: Network & Security / 논리적 통제망]**
    - 예: `pfSense Router & Firewall`, `AWS VPC (172.16.0.0/16)`, `Public/Private Subnets`
    - 두 환경을 잇는 통로(`WireGuard VPN Tunnel`)를 명시적으로 분리해서 표현.
- **[Layer 3: Compute Nodes / 인스턴스 및 노드]**
    - 예: `K8s Control-Plane (6GB)`, `Worker Nodes`, `NAT Gateway Instance`, `AI Server (ml.g5.xlarge)`
    - 노드의 OS나 스펙이 있다면 짧게 기재.
- **[Layer 4: Workloads & Roles / 작업 중심의 내부 컴포넌트]** (가장 중요)
    - 노드 내부에 실제로 떠 있는 핵심 프로세스를 박스로 배치.
    - 단순 이름이 아닌 **[컴포넌트명 : 핵심 역할]** 형태로 명시.
    - 예: `Java Backend (Job 생명주기 관리 및 DB 기록)`, `Go Worker Job (S3 업로드 및 AI 트리거)`, `RabbitMQ (학습 완료 이벤트 큐)`, `Calico-Node (BGP 라우팅)`

# Flow Sequence & Connection Rules (데이터 흐름 규칙)
- 통신 화살표에는 반드시 **[순서 번호]**와 **[프로토콜/포트]**, **[액션 명]**을 라벨링해. (예: `1. HTTPS (443): API 요청`, `2. AMQP (5672): 이벤트 발행`)
- 단방향(->)과 양방향(<->) 통신을 명확히 구분해.

# Output Format Specification
1. **아키텍처 구조 요약 (Architecture Summary)**: 각 레이어별 구성 요소를 마크다운 트리 구조로 요약.
2. **트래픽 시나리오 (Workflow Sequence)**: 사용자 요청부터 AI 학습 완료 후 알림까지의 핵심 트래픽 흐름을 텍스트로 정리.
3. **Diagrams-as-Code (선택: Advanced Mermaid.js 또는 PlantUML C4 Model)**:
    - 복잡한 중첩 구조(Subgraph 내부에 Subgraph)를 완벽히 지원하는 코드로 작성해.
    - 레이어 구분을 위해 색상(Style/Class) 명세를 반드시 포함해.
    - 렌더링 시 노드가 겹치거나 깨지지 않도록 방향(Top to Bottom 또는 Left to Right)을 최적화해.

결과물은  .claude/ 경로에 guide.md 로작성