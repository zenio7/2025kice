Azure AKS 서술형 시험문제 및 모범답안 
문제 1: Pod CrashLoopBackOff 원인과 해결방안

문제: 운영 중인 AKS 클러스터에서 특정 Pod가 CrashLoopBackOff 상태에 빠져있습니다. 이 문제의 가능한 원인들과 각각에 대한 해결 방법을 서술하시오.

모범답안: CrashLoopBackOff는 Pod가 반복적으로 실패하여 재시작되는 상태입니다. 주요 원인과 해결방법은:

    애플리케이션 오류: 잘못된 환경 변수, 누락된 종속성, 코드 버그
        해결: kubectl logs <pod-name> --previous로 로그 확인, 환경 변수 검증

    리소스 제한 초과: CPU/메모리 limits 설정이 너무 낮음
        해결: resources.requests와 resources.limits 값 조정

    ConfigMap/Secret 오류: 필요한 설정 파일이나 환경 변수 누락
        해결: ConfigMap과 Secret 존재 여부 확인, 올바른 마운트 경로 검증

    잘못된 ENTRYPOINT/CMD: 컨테이너 시작 명령어 오류
        해결: Dockerfile 검토, command/args 필드 수정

    종속 서비스 미준비: 데이터베이스 등 외부 서비스 연결 실패
        해결: initContainers 사용, 재시도 로직 구현

문제 2: Pod 스케줄링 분산 설계

문제: 3개의 워커 노드에 5개의 Pod를 배포할 때, 특정 노드에 Pod가 몰리는 것을 방지하고 고가용성을 보장하는 방법을 설계하시오.

모범답안: Pod Anti-Affinity를 사용하여 동일한 애플리케이션의 Pod를 다른 노드에 분산 배치:

spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - myapp
        topologyKey: kubernetes.io/hostname

추가 고려사항:

    preferredDuringScheduling 사용: Pod 수가 노드 수보다 많을 때 유연한 배치
    Pod Topology Spread Constraints: 더 세밀한 분산 제어
    가중치 설정: weight 값으로 우선순위 지정

문제 3: 노드 Scale-in 시 Pod 생명주기 보호

문제: AKS 클러스터 자동 스케일링으로 노드가 축소될 때 실행 중인 애플리케이션의 가용성을 보장하는 방법을 설명하시오.

모범답안: Pod Disruption Budget (PDB) 설정으로 최소 가용 Pod 수 보장:

apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2  # 또는 "50%"
  selector:
    matchLabels:
      app: myapp

추가 보호 메커니즘:

    gracefulTerminationPeriodSeconds: Pod 종료 시 충분한 시간 제공
    preStop hooks: 종료 전 정리 작업 수행
    drain timeout 설정: 노드 드레인 시 대기 시간 조정
    cluster-autoscaler 설정: scale-down-delay-after-add, skip-nodes-with-system-pods

문제 4: Liveness/Readiness Probe 설계

문제: 애플리케이션 시작에 3분이 걸리는 서비스가 있습니다. 이 서비스의 안정적인 배포를 위한 Health Check 설정을 설계하시오.

모범답안:

spec:
  containers:
  - name: myapp
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 200  # 시작 시간 + 버퍼
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 180  # 애플리케이션 준비 시간
      periodSeconds: 5
      successThreshold: 2
      failureThreshold: 3

고려사항:

    Startup Probe 추가: 긴 시작 시간을 위한 별도 프로브
    TCP Socket 프로브: HTTP 엔드포인트가 없을 때 사용
    exec 프로브: 커스텀 스크립트 실행

문제 5: Ingress 500 에러 해결방안

문제: AKS에서 Ingress Controller를 통해 서비스에 접근할 때 500 Internal Server Error가 발생합니다. 이 문제의 원인과 해결 과정을 서술하시오.

모범답안: 500 에러의 주요 원인과 진단 절차:

    Service-Pod 연결 문제:
        확인: kubectl get endpoints <service-name>
        Service selector와 Pod labels 매칭 확인
        targetPort가 컨테이너 포트와 일치하는지 검증

    Pod 준비 상태:
        Readiness probe 실패로 엔드포인트에서 제외됨
        해결: Readiness probe 설정 조정, Pod 상태 확인

    네트워크 정책:
        NetworkPolicy가 트래픽 차단
        NSG(Network Security Group) 규칙 확인

    Ingress 설정 오류:

    annotations:
      nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
      nginx.ingress.kubernetes.io/ssl-redirect: "false"

    디버깅 명령어:

    kubectl logs <ingress-controller-pod> -n ingress-nginx
    kubectl describe ingress <ingress-name>
    kubectl get events --sort-by='.lastTimestamp'

문제 6: Rolling Update 중 서비스 연속성 보장

문제: 배포 업데이트 중 서비스 중단 없이 안전하게 롤링 업데이트를 수행하는 전략을 설계하시오.

모범답안:

spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 추가 Pod 허용
      maxUnavailable: 1  # 동시 중단 Pod 수
  template:
    spec:
      containers:
      - name: app
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]

추가 전략:

    PodDisruptionBudget 설정: minAvailable: 3
    Readiness Probe: 새 Pod가 트래픽 받기 전 준비 확인
    Connection draining: preStop hook으로 기존 연결 처리
    Blue-Green 또는 Canary 배포 고려

이 문제들은 실제 AKS 운영에서 자주 발생하는 이슈들을 다루며, Azure 공식 문서에서 권장하는 모범 사례를 기반으로 작성되었습니다.

 

Azure AKS GPU 환경 서술형 시험문제 및 모범답안
문제 1: Kubernetes QoS Classes와 GPU 워크로드 자원 효율화

문제: AKS 클러스터에서 GPU 워크로드를 실행할 때, Kubernetes QoS(Quality of Service) 클래스가 자원 관리에 미치는 영향을 설명하고, GPU 워크로드에 적합한 QoS 설정 방법을 서술하시오.

모범답안:

1. Kubernetes QoS 클래스 3가지:

Guaranteed (보장형)

    CPU와 메모리의 request와 limit이 동일하게 설정된 경우
    가장 높은 우선순위로 자원 부족시 마지막에 eviction
    GPU 워크로드에 권장되는 설정

resources:
  requests:
    memory: "8Gi"
    cpu: "4"
    nvidia.com/gpu: 1
  limits:
    memory: "8Gi"
    cpu: "4"
    nvidia.com/gpu: 1

Burstable (버스트형)

    request < limit으로 설정되어 여유 자원 사용 가능
    중간 우선순위로 BestEffort 다음 eviction

resources:
  requests:
    memory: "4Gi"
    cpu: "2"
  limits:
    memory: "8Gi"
    cpu: "4"

BestEffort (최선형)

    request와 limit이 설정되지 않은 경우
    가장 낮은 우선순위로 자원 부족시 가장 먼저 eviction
    GPU 워크로드에는 부적합

2. GPU 워크로드 최적화 방안:

    GPU 학습 워크로드: Guaranteed QoS 사용
    GPU 추론 워크로드: 부하에 따라 Burstable 고려
    노드 압박시 GPU Pod 보호를 위해 Guaranteed 설정 필수

문제 2: Multi-Instance GPU (MIG) 구성과 활용

문제: NVIDIA A100 GPU를 사용하는 AKS 클러스터에서 MIG를 활용하여 GPU 자원을 효율적으로 사용하는 방법을 설명하시오. 기존 GPU와 차이점과 구성 방법을 포함하여 서술하시오.

모범답안:

1. MIG 개념과 특징:

    NVIDIA A100, H100 등 최신 GPU에서 지원
    단일 GPU를 최대 7개의 독립적인 인스턴스로 분할
    각 인스턴스는 독립적인 메모리, 캐시, SM(Streaming Multiprocessor) 보유
    하드웨어 수준의 격리로 fault isolation 제공

2. 기존 GPU와의 차이점:

기존 GPU (T4, V100 등):

    GPU 전체를 단일 Pod가 독점
    Time-slicing으로만 공유 가능 (소프트웨어 레벨)
    자원 격리 불완전

MIG 지원 GPU (A100, H100):

    하드웨어 레벨 파티셔닝
    QoS 보장
    동시 다중 워크로드 실행

3. AKS에서 MIG 구성:

노드 풀 생성시 GPU 인스턴스 프로파일 지정:

az aks nodepool add \
  --name migpool \
  --cluster-name myAKSCluster \
  --resource-group myResourceGroup \
  --node-vm-size Standard_ND96asr_v4 \
  --gpu-instance-profile MIG1g \
  --node-count 1

4. MIG 파티션 크기 옵션 (A100 40GB 기준):

    1g.5gb: 1개 컴퓨트 유닛, 5GB 메모리 (최대 7개)
    2g.10gb: 2개 컴퓨트 유닛, 10GB 메모리 (최대 3개)
    3g.20gb: 3개 컴퓨트 유닛, 20GB 메모리 (최대 2개)
    7g.40gb: 전체 GPU 사용 (1개)

문제 3: GPU 노드 풀 구성과 Pod 스케줄링

문제: GPU와 non-GPU 워크로드가 혼재된 AKS 클러스터에서 GPU 자원을 효율적으로 사용하기 위한 노드 풀 구성 전략과 Pod 스케줄링 방법을 서술하시오.

모범답안:

1. 노드 풀 분리 전략:

GPU 노드 풀 생성 및 Taint 설정:

az aks nodepool add \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster \
  --name gpupool \
  --node-count 1 \
  --node-vm-size Standard_NC6s_v3 \
  --node-taints sku=gpu:NoSchedule \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 3

2. Pod 스케줄링 설정:

GPU Pod에 Toleration 추가:

spec:
  tolerations:
  - key: "sku"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  
  nodeSelector:
    hardware: gpu
    
  containers:
  - name: gpu-app
    resources:
      limits:
        nvidia.com/gpu: 1

3. GPU 자원 효율화 방안:

    Node Affinity: GPU 노드에만 스케줄링
    Pod Anti-Affinity: GPU Pod 분산 배치
    Priority Classes: GPU Pod에 높은 우선순위 부여
    Resource Quotas: 네임스페이스별 GPU 할당량 제한

문제 4: GPU Driver 관리와 NVIDIA GPU Operator

문제: AKS에서 GPU 드라이버 설치 방법과 NVIDIA GPU Operator 사용시 고려사항을 설명하시오.

모범답안:

1. GPU 드라이버 설치 옵션:

자동 설치 (기본값):

    AKS가 자동으로 NVIDIA 드라이버 설치
    노드 풀 생성시 자동 적용

수동 설치 (--gpu-driver none):

az aks nodepool add \
  --name gpupool \
  --gpu-driver none \
  --node-vm-size Standard_NC6s_v3

    NVIDIA GPU Operator 사용시 선택
    커스텀 드라이버 버전 필요시

2. NVIDIA GPU Operator 배포:

helm install gpu-operator nvidia/gpu-operator \
  -n gpu-operator --create-namespace \
  --set driver.enabled=false \
  --set toolkit.enabled=false

3. GPU Operator 장점:

    DCGM으로 메트릭 수집
    런타임 MIG 프로파일 변경 가능
    자동 검증 컨테이너 실행

문제 5: GPU 워크로드 모니터링과 최적화

문제: AKS GPU 클러스터에서 자원 사용률을 모니터링하고 최적화하는 방법을 서술하시오.

모범답안:

1. GPU 메트릭 수집:

# GPU 사용률 확인
kubectl exec -it <pod-name> -- nvidia-smi

# Node의 GPU 할당 상태
kubectl describe node <gpu-node-name> | grep nvidia

2. 자원 최적화 전략:

MIG 활용한 세분화:

    소규모 추론: 1g.5gb 인스턴스
    중규모 학습: 3g.20gb 인스턴스
    대규모 학습: 7g.40gb 전체 GPU

3. 비용 최적화:

    Spot 인스턴스 활용 (내결함성 워크로드)
    Cluster Autoscaler로 동적 스케일링
    유휴 시간 최소화를 위한 Job 스케줄링

4. 성능 모니터링 지표:

    GPU 사용률 (%)
    GPU 메모리 사용량
    SM 활용도
    온도 및 전력 소비

이러한 구성을 통해 AKS에서 GPU 자원을 효율적으로 활용하면서도 워크로드별 요구사항을 충족시킬 수 있습니다.



2. RAG를 위한 저장소/DBMS Azure 서비스 선택관련 개념: RAG 구현을 위한 Azure의 벡터 데이터베이스 옵션으로 Azure Database for PostgreSQL (pgvector 확장), Azure Cognitive Search (벡터 검색), Azure Cosmos DB (통합 벡터 데이터베이스) 등이 있습니다.

공식 문서 링크:

    Vector search on Azure Database for PostgreSQL with pgvector
    Vector search - Azure AI Search
    Integrated vector database - Azure Cosmos DB

핵심 키워드: pgvector, Azure Cognitive Search, vector embedding, Azure OpenAI, hybrid search, DiskANN, HNSW, IVFFlat
3. Azure Function Tier 특징이해와 선택(대용량 Batch를 위한 Function Tier)관련 개념: Azure Functions의 3가지 주요 호스팅 플랜 - Consumption Plan, Premium Plan(Elastic Premium), Dedicated Plan(App Service)과 Durable Functions

공식 문서 링크:

    Azure Functions Premium plan
    Azure Functions scale and hosting
    Performance and scale in Durable Functions

핵심 키워드:

    Consumption Plan: 10분 타임아웃, 자동 스케일, 콜드 스타트 발생
    Premium Plan(EP1, EP2, EP3): 30분 기본 타임아웃(무제한 설정 가능), Always Ready 인스턴스, Prewarmed 인스턴스, VNet 통합
    Dedicated Plan: Always On 설정 필요, 수동 스케일
    Durable Functions: 대용량 배치 처리, 체크포인트 및 재생 기능

4. Azure Disk 관리 운영 상의 특징: 크기 조절방법 및 제약 사항, OS Reboot 필요 등관련 개념: Azure Managed Disk의 크기 조정 방법, 제약사항, VM 재부팅 필요성, OS 디스크와 데이터 디스크의 차이

공식 문서 링크:

    Expand Virtual Hard Disks on Windows VM
    Expand Virtual Hard Disks on Linux VM
    Change the performance of Azure managed disks

핵심 키워드:

    Expand without downtime: 데이터 디스크만 지원, OS 디스크는 불가
    4 TiB 제한: 4TiB 이하 디스크는 VM deallocate 필요
    제약사항: Ultra Disk, Premium SSD v2, Shared Disk 미지원
    API 버전: 2021-04-01 이상 필요
    Classic VM: 일부 SKU에서 미지원
    Detach/Attach: 데이터 디스크는 Hot Remove 가능

5. Terraform 내용 이해관련 개념: Terraform azurerm provider의 Load Balancer 리소스 구문과 Azure Portal 용어 차이점

공식 문서 링크:

    Terraform - Create public load balancer
    Terraform - Create internal load balancer

핵심 키워드/차이점:

    Terraform: azurerm_lb, azurerm_lb_backend_address_pool, azurerm_lb_probe, azurerm_lb_rule
    SKU: Terraform에서 sku = "Standard" (Portal의 "Standard SKU")
    Frontend IP: frontend_ip_configuration (Portal의 "Frontend IP configurations")
    Backend Pool: azurerm_lb_backend_address_pool (Portal의 "Backend pools")
    Health Probe: azurerm_lb_probe (Portal의 "Health probes")
    Load Balancing Rule: azurerm_lb_rule (Portal의 "Load balancing rules")

6. 비용 최적화 방안: Saving Plan, RI, Size 조절, VM Schedule 방안에 대한 이해와 적용법관련 개념: Azure 비용 최적화를 위한 Reserved Instances, Savings Plans, VM 크기 조정, VM 일정 관리

공식 문서 링크:

    Decide between a savings plan and a reservation
    What are Azure Reservations?
    Virtual machine size flexibility

핵심 키워드 및 비용 효율적인 순서:

    Reserved Instances (RI): 최대 72% 절약, 특정 VM 크기/지역 고정, 안정적 워크로드에 최적
    Savings Plans: 최대 65% 절약, 시간당 금액 약정, 유연한 적용 (모든 컴퓨팅 서비스)
    Instance Size Flexibility: RI 내에서 동일 인스턴스 패밀리 그룹 내 크기 변경 가능
    VM Rightsizing: 사용률 분석 후 적절한 VM 크기로 조정
    Auto-shutdown/Start-stop schedule: 비사용 시간대 VM 자동 중지로 비용 절감

비용 효율 순서: RI (장기 안정적) > Savings Plan (유연성 필요) > VM 크기 최적화 > 일정 관리
7. PV, PVC 운영상의 특징(Azure File)관련 개념: AKS의 PV/PVC와 Azure Files 통합, Private Endpoint 구성, SMB/NFS 프로토콜 차이점

공식 문서 링크:

    Create a persistent volume with Azure Files in AKS
    Use Container Storage Interface (CSI) driver for Azure Files
    NFS file shares in Azure Files
    Networking Considerations for Azure Files

핵심 키워드:

    Private Endpoint 설정: networkEndpointType: privateEndpoint 파라미터 사용
    Public 연결 방법: Service Endpoint 또는 공용 IP (SMB 3.x with encryption)
    SMB 프로토콜:
        사용자 기반 인증 (NTLM v2)
        Port 445 사용
        암호화 지원 (SMB 3.x)
    NFS 프로토콜:
        네트워크 수준 인증만 지원
        Private Endpoint 또는 Service Endpoint 필수
        Premium 계정만 지원 (최소 100GiB)
        Mount 옵션: nconnect=4, noresvport, actimeo=30

8. Express Route 개념 및 SKU Tier 특징 이해관련 개념: Azure ExpressRoute의 3가지 SKU Tier (Local, Standard, Premium)의 특징과 차이점

공식 문서 링크:

    FAQ - Azure ExpressRoute
    Azure ExpressRoute Overview
    Plan to manage costs for Azure ExpressRoute

SKU Tier별 특징:

    Local SKU:
        동일 메트로 지역의 Azure 리전만 액세스 가능
        데이터 전송료 무료 (Unlimited data plan 포함)
        가장 비용 효율적
        예: 런던 피어링 위치 → UK South 리전만 액세스

    Standard SKU:
        동일 지정학적 지역 내 모든 Azure 리전 액세스
        Metered 또는 Unlimited 데이터 플랜 선택 가능
        라우트 제한: 4,000개 (private peering)
        예: 유럽 피어링 → 유럽 내 모든 Azure 리전 액세스

    Premium SKU:
        전 세계 모든 Azure 리전 글로벌 액세스
        라우트 제한 증가: 10,000개 (private peering)
        VNet 링크 수 증가
        추가 회로 요금 발생
        Global Reach Add-on 지원

10. Istio Service Mesh 개념과 각 자원/서비스 용어와 이해관련 개념: Istio Service Mesh의 주요 구성 요소와 용어

공식 문서 링크:

    Istio Traffic Management
    Virtual Service
    Gateway
    Destination Rule

핵심 용어 및 개념:

    Gateway:
        메시 엣지에서 인바운드/아웃바운드 트래픽 관리
        Ingress/Egress 트래픽 제어
        독립형 Envoy 프록시에 적용

    Virtual Service:
        트래픽 라우팅 규칙 정의
        hosts, routing rules, match conditions 설정
        Canary 배포, A/B 테스트 지원

    Destination Rule:
        라우팅 후 트래픽 정책 적용
        로드밸런싱 정책 (ROUND_ROBIN, LEAST_REQUEST 등)
        서브셋(subset) 정의 및 버전 관리
        Connection pool, Circuit breaker 설정

    Service Entry:
        외부 서비스를 메시에 추가
        메시 외부 종속성 관리

    Sidecar:
        Envoy 프록시 범위 구성
        네임스페이스 격리

트래픽 흐름: Request → Gateway → VirtualService → DestinationRule → Pod
11. AKS 성능 개선에 대한 이해관련 개념: AKS 성능 최적화를 위한 컨테이너 크기 조정, 리소스 제한, 워커 노드 분산 전략

공식 문서 링크:

    Performance and scaling best practices for small to medium workloads
    Performance and scaling best practices for large workloads
    Resource management best practices

핵심 최적화 방법:

    컨테이너 크기 조정:
        Resource requests: CPU 100m, Memory 128Mi (최소 요구사항)
        Resource limits: CPU 250m, Memory 256Mi (최대 사용량)
        CPU 단위: 1.0 = 1 vCPU core, 100m = 0.1 vCPU

    노드 크기 선택:
        프로덕션: Standard_D2s_v3, Standard_D4s_v3 (SSD 지원)
        메모리 집약적: Standard_E2s_v3, Standard_E4s_v3
        시스템 노드풀: Standard_D16ds_v5 이상
        Ephemeral OS 디스크 사용 권장

    워커 노드 분산:
        노드풀당 최대 1,000개 노드
        5,000 노드 클러스터: 최소 5개 사용자 노드풀
        스케일링: 현재 규모의 10-20% 단위로 증가

    오토스케일링:
        HPA (Horizontal Pod Autoscaler): 단기 피크 대응
        CA (Cluster Autoscaler): 장기 피크 대응
        VPA (Vertical Pod Autoscaler): 리소스 자동 조정
        Karpenter: 더 효율적인 노드 관리

    네트워킹 최적화:
        Azure CNI 사용 (Kubenet 대비 성능 향상)
        Accelerated Networking 활성화
        750 노드 이하에서 내부 로드밸런서 생성

12. Node Pool 개념 이해와 운영 방안 특징관련 개념: AKS System Node Pool과 User Node Pool의 차이점, 전용 노드 사용 가이드라인

공식 문서 링크:

    Use system node pools in AKS
    Manage node pools in AKS
    Create node pools in AKS

핵심 특징 및 차이점:
System Node Pool:

    목적: 시스템 핵심 Pod 호스팅 (CoreDNS, tunnelfront 등)
    제한사항:
        최소 1개 노드 필수 (권장: 3개)
        Linux OS만 지원
        최소 2 vCPU, 4GB 메모리
        최소 30개 Pod 지원 필요
        Spot VM 사용 불가
        B-series(Burstable) VM 권장하지 않음
    레이블: kubernetes.azure.com/mode: system
    Taint 설정: CriticalAddonsOnly=true:NoSchedule (애플리케이션 Pod 차단)

User Node Pool:

    목적: 애플리케이션 워크로드 실행
    특징:
        Linux/Windows 모두 지원
        0개 노드까지 스케일 다운 가능
        삭제 가능
        Spot VM 사용 가능
        System pool로 전환 가능

공통 자원 운영 가이드라인:

    전용 System Node Pool 사용: 공통 자원(시스템 구성요소)는 전용 시스템 노드풀에 배치
    격리 원칙: 시스템 Pod와 애플리케이션 Pod 분리
    DaemonSet 예외: kube-proxy 같은 DaemonSet은 모든 노드에 배포됨

모드 전환 명령어:

# User pool을 System pool로 변경
az aks nodepool update --mode system

# System pool을 User pool로 변경 (다른 System pool 존재 시)
az aks nodepool update --mode user

13. CICD Pipeline 구성시 Credential관리 주의 사항에 대한 이해와 방법

이제 나머지 문제들을 차례대로 정리하겠습니다.관련 개념: CI/CD 파이프라인에서 자격 증명 관리 시 주의사항과 Azure Key Vault 활용

공식 문서 링크:

    Use Azure Key Vault secrets in Azure Pipelines
    Best practices for using Azure Key Vault
    Protect secrets in Azure Pipelines

핵심 주의사항 및 방법:

    Key Vault 격리 원칙:
        애플리케이션별, 환경별(Dev/Test/Prod), 리전별로 별도 Key Vault 생성
        비밀 공유 방지 및 침해 시 영향 범위 제한

    Service Principal 관리:
        Workload Identity Federation 사용 (권장)
        최소 권한 원칙: Get, List 권한만 부여
        정기적인 자격 증명 순환(Rotation)

    접근 제어:
        RBAC(역할 기반 접근 제어) 사용
        Managed Identity 활용 (가장 안전)
        Purge Protection 활성화

    파이프라인 보안:
        Variable Groups에 Key Vault 연결
        비밀을 평문으로 .yml 파일에 저장 금지
        콘솔에 비밀 출력 방지

    네트워크 보안:
        Private Endpoint 사용
        특정 IP 주소만 허용하는 방화벽 규칙 설정
        Service Endpoint 구성

    모니터링:
        Key Vault 로깅 활성화
        Azure Monitor로 모든 비밀 활동 추적
        Azure Event Grid로 생명주기 모니터링

14. Github Action의 기본구성 이해와 기본 Task명에 대한 이해관련 개념: GitHub Actions의 기본 구성 요소와 Azure 배포 관련 주요 Task

공식 문서 링크:

    What is GitHub Actions for Azure?
    GitHub Actions for Azure
    Deploy by Using GitHub Actions

기본 구성 요소:

    Workflow (워크플로우):
        .github/workflows/ 디렉토리의 YAML 파일
        자동화된 프로세스 정의
        여러 Job으로 구성

    Events (이벤트):
        push, pull_request, schedule, workflow_dispatch
        워크플로우 트리거 조건

    Jobs (작업):
        병렬 실행 (기본값)
        needs로 의존성 설정 가능
        Runner에서 실행

    Steps (단계):
        Job 내의 순차적 작업
        Actions 또는 쉘 명령어 실행

    Actions (액션):
        재사용 가능한 작업 단위
        GitHub Marketplace에서 제공

주요 Azure 관련 Actions/Tasks:

    인증:
        azure/login@v2: Azure 로그인 (OIDC, Service Principal)
        azure/get-keyvault-secrets: Key Vault 비밀 가져오기

    App Service:
        azure/webapps-deploy: 웹앱 배포
        azure/functions-action: Functions 배포

    컨테이너:
        azure/aks-set-context: AKS 컨텍스트 설정
        azure/docker-login: ACR 로그인
        azure/k8s-deploy: Kubernetes 배포

    인프라:
        azure/arm-deploy: ARM 템플릿 배포
        azure/CLI: Azure CLI 실행
        azure/powershell: Azure PowerShell 실행

    언어별 설정:
        actions/setup-node: Node.js 설정
        actions/setup-dotnet: .NET 설정
        actions/setup-java: Java 설정
        actions/setup-python: Python 설정

워크플로우 예제 구조:

name: Deploy to Azure
on: [push]
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
      - uses: azure/webapps-deploy@v3

15. Storage의 암호화 방식에 대한 이해관련 개념: Azure Storage 암호화 방식 - SSE, 인프라 암호화, 이중 암호화

공식 문서 링크:

    Azure Storage encryption for data at rest
    Server-side encryption of Azure managed disks
    Customer-managed keys for account encryption

핵심 암호화 방식:
1. Server-Side Encryption (SSE):

    기본 활성화: 모든 스토리지 계정에 자동 적용
    암호화 알고리즘: 256-bit AES 암호화
    FIPS 140-2 준수
    투명한 암호화/복호화: 성능 영향 없음

2. 키 관리 옵션:

    Microsoft-managed keys (기본값): Microsoft가 자동 관리 및 순환
    Customer-managed keys (CMK): Azure Key Vault에 저장, 고객이 직접 관리
    Customer-provided keys: Blob Storage 작업 시 요청마다 제공

3. Double Encryption (이중 암호화):

    서비스 레벨 암호화: 첫 번째 계층
    인프라 레벨 암호화: 두 번째 계층
    다른 암호화 알고리즘과 키 사용
    Ultra Disk, Premium SSD v2는 미지원

4. Encryption Scopes:

    컨테이너 또는 개별 Blob 레벨 암호화
    동일 스토리지 계정 내 보안 경계 생성

5. 특수 암호화 유형:

    Encryption at Host: VM 호스트에서 암호화 (임시 디스크 포함)
    Azure Disk Encryption (ADE): BitLocker(Windows)/DM-Crypt(Linux) 사용
    Envelope Encryption: DEK를 KEK로 보호

잘못된 옵션 (함정):

    DB 컬럼 암호화: TDE(Transparent Data Encryption)나 CLE(Cell-level Encryption)은 스토리지 암호화와 별개
    클라이언트 사이드 암호화: 스토리지 서비스 암호화와 다른 개념

16. Azure Native 보안 서비스 이해

이제 남은 문제들을 계속 정리하겠습니다.관련 개념: Azure Native 보안 서비스 종류와 기능

공식 문서 링크:

    Microsoft Defender for Cloud
    Azure DDoS Protection Overview
    Microsoft Sentinel

주요 Azure Native 보안 서비스:
1. Microsoft Defender for Cloud (구 Azure Security Center + Azure Defender):

    CSPM: Cloud Security Posture Management
    CWPP: Cloud Workload Protection Platform
    보호 대상: Servers, App Service, Storage, SQL, Key Vault, Resource Manager, DNS, Kubernetes, Container Registry
    기능: 보안 점수, 취약점 평가, JIT VM 접근, 적응형 애플리케이션 제어

2. Microsoft Sentinel:

    SIEM: Security Information Event Management
    SOAR: Security Orchestration Automated Response
    기능: AI 기반 위협 탐지, 보안 분석, 위협 헌팅, 자동화된 대응

3. Azure Key Vault:

    비밀 관리: 암호, 인증서, API 키 저장
    HSM: Hardware Security Module 지원
    FIPS 140-2 Level 2/3 준수

4. Azure DDoS Protection:

    Basic: 기본 인프라 수준 보호 (무료)
    Network Protection: Layer 3/4 고급 보호
    IP Protection: Pay-per-protected IP 모델
    DDoS Rapid Response 팀 지원

5. Azure Web Application Firewall (WAF):

    Layer 7 보호: SQL Injection, XSS, OWASP Top 10
    통합 서비스: Application Gateway, Front Door, CDN
    OWASP ModSecurity Core Rule Set

6. Azure Firewall:

    네트워크 보안: Stateful 방화벽 서비스
    위협 인텔리전스 기반 필터링
    고가용성 및 무제한 클라우드 확장성

7. Azure Active Directory (Microsoft Entra ID):

    Identity Management: SSO, MFA, 조건부 액세스
    Privileged Identity Management (PIM)
    Identity Protection

8. Azure Policy:

    거버넌스: 조직 표준 관리 및 적용
    규정 준수 평가
    Policy as Code

9. Azure Network Security Groups (NSG):

    가상 네트워크 보안
    인바운드/아웃바운드 트래픽 규칙

10. Azure Information Protection (AIP):

    데이터 분류 및 레이블링
    문서 및 이메일 보호

11. Azure Bastion:

    안전한 RDP/SSH 연결
    Public IP 없이 VM 접근

12. Microsoft Antimalware:

    실시간 보호
    바이러스, 스파이웨어 차단

17. Azure Monitoring 서비스 활용시 비용적인 측면의 Log Data 관리 방법에 대한 이해관련 개념: Azure Monitor/Log Analytics의 로그 데이터 비용 최적화 방법

공식 문서 링크:

    Azure Monitor Logs cost calculations and options
    Azure Monitor cost and usage
    Pricing - Azure Monitor

비용 효율적인 로그 데이터 관리 방법:
1. Commitment Tiers (약정 계층):

    100 GB/day부터 시작: 15% 할인
    최대 50 TB/day: 36% 할인
    Pay-As-You-Go 대비 최대 52% 절약
    언제든지 업그레이드 가능, 31일 후 다운그레이드 가능

2. Table Plans (테이블 계획):

    Analytics Logs: 표준 쿼리 및 분석용
    Basic Logs: 제한된 쿼리, 저렴한 요금 (최대 1/10 비용)
    Auxiliary Logs: 규정 준수용 장기 보관

3. Data Retention 정책:

    Interactive Retention: 최대 2년, 즉시 쿼리 가능
    Archive: 최대 12년, 저렴한 비용으로 장기 보관
    테이블별 개별 보존 기간 설정 가능

4. 비용 절감 기법:

    데이터 필터링: 불필요한 로그 제외
    샘플링: Application Insights에서 원격 측정 볼륨 감소
    Data Collection Rules (DCR): 정교한 로그 수집 제어
    일일 한도 설정: 예상치 못한 비용 방지

5. Storage Tiers 활용 (Blob Storage):

    Hot Tier: 자주 액세스 (30-90일)
    Cool Tier: 가끔 액세스 (90-365일)
    Archive Tier: 장기 보관 (1년 이상)
    자동 계층 전환: Lifecycle Management Rules

6. 무료 데이터 수집:

    _IsBillable = false 테이블: AzureActivity, Heartbeat, Usage, Operation
    Microsoft Defender 특정 보안 테이블
    첫 31일 보존 기간 무료

7. Dedicated Clusters:

    여러 워크스페이스 통합: 더 큰 Commitment Tier 활용
    500 GB/day 이상 시 고려

8. 모니터링 및 최적화 도구:

    Azure Cost Management + Billing: 비용 추적
    Usage and Estimated Costs: 최적 약정 계층 추천
    KQL 쿼리: 데이터 볼륨 분석

9. 대안 솔루션 (대용량 로그):

    Azure Event Hubs + ELK Stack
    Azure Data Explorer: 고급 로그 분석
    Azure Sentinel Data Lake: 대용량 보안 로그 장기 보관

Azure Cloud Architecture 시험 힌트에 대해 각 문제별로 관련 개념과 공식 문서를 찾아 정리해드리겠습니다.Azure Cloud Architecture 시험에 대한 힌트를 바탕으로 각 문제별로 관련 개념과 공식 문서를 정리해드리겠습니다.
18. Migration From VMWare VM: 제약조건, 구성법 등

관련 개념:

    Azure Migrate를 통한 VMware 마이그레이션
    데이터 플랫폼 구축 시 기술적 요구사항
    Agentless vs Agent-based 마이그레이션

공식 문서 링크:

    Azure Migrate appliance requirements
    VMware vSphere VM migration support matrix
    Migrate VMware VMs agentless - Azure Migrate

정답 키워드:

    Azure VMware Solution (AVS)
    데이터 플랫폼의 스케일링, 탄력성, 가용성 특징
    Veeam Data Platform을 통한 크로스 플랫폼 복원

19. PV/PVC with Azure File 구성시 필요한 Network 설정

관련 개념:

    Azure Files와 Private Endpoint 구성
    AKS에서 Persistent Volume Claims (PVC) 사용
    네트워크 격리 클러스터에서의 스토리지 구성

공식 문서 링크:

    Create a persistent volume with Azure Files in AKS
    Configure Azure Files Network Endpoints
    Use private endpoints - Azure Storage

정답 키워드:

    networkEndpointType: privateEndpoint 스토리지 클래스 설정
    Private Endpoint, Private DNS Zone 구성
    스토리지 계정에 대한 Private Endpoint 생성 필수

20. Data Platform 구축시 요구사항정의와 그에 따른 Cloud특징

관련 개념:

    현대적 데이터 플랫폼 아키텍처
    클라우드 네이티브 데이터 솔루션
    Azure 데이터 서비스들의 특징

공식 문서 링크:

    Modern Data Platform Architecture for SMBs
    Create a Modern Analytics Architecture using Azure Databricks
    Analytics end-to-end with Azure Synapse

정답 키워드:

    확장성(Scalability), 탄력성(Elasticity), 가용성(High Availability)
    통합된 데이터 분석 플랫폼
    배치 처리, 실시간 처리, 데이터 레이크 아키텍처

21. 구독 할당 및 관리에 대한 이해

관련 개념:

    Azure RBAC (Role-Based Access Control)
    구독 레벨 권한 관리
    정책 기반 액세스 제어

공식 문서 링크:

    What is Azure role-based access control (Azure RBAC)?
    Assign Azure roles using the Azure portal
    Azure built-in roles

정답 키워드:

    보안 주체(Security Principal), 역할 정의(Role Definition), 범위(Scope)
    구독 소유자, 기여자, 읽기 권한자 역할
    Microsoft.Authorization/roleAssignments/write 권한

22. VNet Flow log 서비스 종류와 특징 이행

관련 개념:

    Azure Virtual Network Flow Logs (새로운 서비스)
    기존 NSG Flow Logs와의 차이점
    Network Watcher 기능 확장

공식 문서 링크:

    Virtual Network Flow Logs - Azure Network Watcher
    NSG Flow Logs Overview
    Manage virtual network flow logs

정답 키워드:

    VNet Flow Logs: 가상 네트워크 레벨 로깅, 암호화 상태 모니터링
    NSG Flow Logs: 네트워크 보안 그룹 레벨 로깅 (2027년 9월 30일 은퇴 예정)
    향상된 네트워크 가시성, Traffic Analytics 지원


1. Azure Resource RBAC Policy: 관리그룹 및 계층형 구조에 따른 사용자권한 제어
관련 개념 정리

    RBAC (Role-Based Access Control): Azure 리소스에 대한 세분화된 액세스 관리 시스템
    권한 상속: 관리 그룹 → 구독 → 리소스 그룹 → 리소스 순으로 권한이 상속됨
    주요 Built-in 역할: Owner(소유자), Contributor(기여자), Reader(읽기 권한자), User Access Administrator(사용자 액세스 관리자)
    권한 할당 구조: Security Principal(보안 주체) + Role Definition(역할 정의) + Scope(범위)

공식 문서 링크

    Azure RBAC 개요
    Azure 기본 제공 역할
    관리 그룹으로 리소스 구성

예상 키워드

    Owner, Contributor, Reader
    권한 상속(Inheritance)
    Deny assignments
    Management Group 계층 구조

2. Azure Storage Account(Object Storage) Tier 별 특성
관련 개념 정리

    액세스 계층: Hot(자주 액세스), Cool(가끔 액세스, 30일 이상 저장), Archive(거의 액세스 안함, 180일 이상 저장)
    Data Lake Storage Gen2: Blob Storage + 계층적 네임스페이스(HNS) 활성화
    제약사항: Archive 계층은 온라인으로 읽기 불가, 리하이드레이션 필요 (표준: 최대 15시간, 높은 우선순위: 1시간 이내)
    CRUD 차이: Archive는 직접 읽기/업데이트 불가, Cool은 조기 삭제 수수료(30일 이내)

공식 문서 링크

    Blob Storage 액세스 계층
    Azure Data Lake Storage Gen2
    액세스 계층 간 성능 및 가격 비교

예상 키워드

    Hot, Cool, Archive
    Hierarchical Namespace (HNS)
    Rehydration
    Early deletion penalty

3. Azure Storage Account Life Cycle Event 활용법
관련 개념 정리

    수명 주기 관리 정책: 마지막 액세스/수정 시간 기준으로 자동 계층 전환 또는 삭제
    Event Grid vs Event Hub:
        Event Grid: 이벤트 기반 아키텍처, HTTP 웹훅, 서버리스 시나리오
        Event Hub: 대용량 스트리밍, 실시간 분석, 로그 집계
    자동화 트리거: Blob 생성/삭제/수정 이벤트를 Logic Apps, Functions와 연결

공식 문서 링크

    Blob Storage 수명 주기 관리
    Event Grid와 Blob Storage
    Event Grid vs Event Hub 비교

예상 키워드

    Lifecycle management policy
    Event Grid (이벤트 라우팅)
    Event Hub (스트리밍)
    Last Modified Time, Last Access Time

4. VNet 구성시 Inbound(LB), Outbound(NAT) 구성방법
관련 개념 정리

    Load Balancer:
        Standard LB: 영역 중복, 가용성 영역 지원, HA 포트
        Basic LB: 무료, 제한적 기능
    NAT Gateway: 아웃바운드 인터넷 연결용, SNAT 포트 고갈 방지
    Application Gateway: L7 로드밸런서, WAF, SSL 종료
    Network Flow: Internet → LB/App GW → VM/VMSS → NAT Gateway → Internet

공식 문서 링크

    Azure Load Balancer 개요
    Virtual Network NAT
    Application Gateway 개요

예상 키워드

    Standard Load Balancer
    NAT Gateway
    SNAT port exhaustion
    Backend pool, Health probe

5. Saving Plan, Reserved Instance을 활용한 비용절감
관련 개념 정리

    Reserved Instance (RI): 1년 또는 3년 약정, 최대 72% 할인, VM 전용
    Savings Plan: 시간당 지출 약정, 더 유연함, 컴퓨팅 리소스 전반 적용
    적용 가능 리소스: VM, App Service, Container Instances, Functions Premium
    적용 불가 리소스: 스토리지, 네트워킹, 데이터베이스(별도 예약 필요)

공식 문서 링크

    Azure 예약 개요
    Azure Savings Plans
    예약 할인 적용 방법

예상 키워드

    1-year, 3-year commitment
    Compute Savings Plan
    Instance flexibility
    Scope (Shared, Single subscription, Resource group)

6. Azure Storage Account 특징과 차이
관련 개념 정리

    중복성 옵션:
        LRS (Locally Redundant): 단일 데이터센터 3개 복사본
        ZRS (Zone Redundant): 3개 가용성 영역
        GRS (Geo-Redundant): 주 지역 LRS + 보조 지역 LRS
        GZRS: 주 지역 ZRS + 보조 지역 LRS
    성능 계층: Standard (HDD), Premium (SSD)
    계정 종류: StorageV2, BlockBlobStorage, FileStorage, BlobStorage(레거시)

공식 문서 링크

    Storage 계정 개요
    Azure Storage 중복성
    Storage 계정 유형 비교

예상 키워드

    LRS, ZRS, GRS, GZRS, RA-GRS
    General-purpose v2
    Premium performance
    99.999999999% (11 9's) durability

7. VMSS 특성에 대한 이해
관련 개념 정리

    자동 크기 조정: CPU, 메모리, 네트워크 메트릭 기반 또는 일정 기반
    최신 동향: Flexible orchestration mode (VM과 VMSS 혼합 관리)
    Load Balancer 통합: 자동으로 백엔드 풀에 추가/제거
    주의사항: 상태 저장 앱은 부적합, 업데이트 도메인과 장애 도메인 고려

공식 문서 링크

    Virtual Machine Scale Sets 개요
    VMSS 자동 크기 조정
    Flexible orchestration

예상 키워드

    Uniform vs Flexible orchestration
    Autoscale rules
    Update domains, Fault domains
    Instance protection

8. DNS Resolver 구성법
관련 개념 정리

    Private DNS Zone: VNet 내부 DNS 해석
    DNS Private Resolver: 온프레미스와 Azure 간 DNS 쿼리 전달
    Conditional Forwarding: 특정 도메인을 특정 DNS 서버로 전달
    필요 포트: DNS (TCP/UDP 53), VPN/ExpressRoute 연결 필요

공식 문서 링크

    Azure DNS Private Resolver
    Private DNS zones
    하이브리드 DNS 해석

예상 키워드

    Inbound endpoint, Outbound endpoint
    Conditional forwarding rules
    Virtual network links
    Port 53 (TCP/UDP)

9. AKS Monitoring를 활용한 자원/Workload 모니터링
관련 개념 정리

    Container Insights: 컨테이너 로그 및 메트릭 수집
    모니터링 구성 요소:
        Node/Pod 메트릭: CPU, 메모리 사용률
        Container 로그: stdout, stderr
        Network 모니터링: 네트워크 성능 및 연결
    Azure Monitor 활성화 시: Log Analytics 작업 영역 생성, Metrics API 활성화

공식 문서 링크

    AKS용 Azure Monitor
    Container Insights 개요
    AKS 진단 설정

예상 키워드

    Container Insights
    Log Analytics workspace
    Prometheus metrics
    Diagnostic settings

10. Azure 기반 Messaging 구성법
관련 개념 정리

    Event Grid: 이벤트 라우팅, Pub/Sub, 서버리스, HTTP 기반
    Event Hub: 빅데이터 스트리밍, 초당 수백만 이벤트, Kafka 호환
    Service Bus: 엔터프라이즈 메시징, 큐/토픽, 트랜잭션, FIFO 보장
    Relay: 하이브리드 연결, 온프레미스와 클라우드 간 보안 통신

공식 문서 링크

    메시징 서비스 비교
    Event Grid 개요
    Service Bus 메시징

예상 키워드

    Event-driven architecture
    Queue vs Topic
    At-least-once delivery
    Hybrid Connections

11. AKS Autoscale 이해: POD, Node(Node Group)
관련 개념 정리

    HPA (Horizontal Pod Autoscaler): CPU/메모리 기반 Pod 자동 확장, 최소 10초마다 체크
    VPA (Vertical Pod Autoscaler): Pod 리소스 요청/제한 자동 조정
    Cluster Autoscaler: Node 부족 시 Node Pool 자동 확장, Pending Pod 감지
    스케일링 순서: HPA가 먼저 Pod 증가 → Pending 상태 → Cluster Autoscaler가 Node 추가

공식 문서 링크

    AKS 클러스터 자동 크기 조정
    HPA를 사용한 Pod 크기 조정
    AKS 크기 조정 개념

예상 키워드

    Horizontal Pod Autoscaler (HPA)
    Cluster Autoscaler
    Pending pods
    Scale-up/Scale-down delay

12. Node Pool 개념 이해와 운영 방안 특징

1. 관련 개념 정리:

    AKS는 System Node Pool과 User Node Pool 두 가지 유형 제공
    System Node Pool: 핵심 시스템 포드(CoreDNS, tunnelfront 등) 실행용
    User Node Pool: 애플리케이션 워크로드 실행용
    공통 리소스나 중요 서비스는 전용 노드 풀 사용 권장

2. 공식문서 링크: 

- https://learn.microsoft.com/en-us/azure/aks/use-system-pools

    https://learn.microsoft.com/en-us/azure/aks/create-node-pools
    https://learn.microsoft.com/en-us/azure/aks/manage-node-pools

3. 정답 키워드:

    System node pool / User node pool
    CoreDNS, tunnelfront, konnectivity
    CriticalAddonsOnly=true:NoSchedule (시스템 풀 전용 설정)
    kubernetes.azure.com/mode: system (레이블)

13. CICD Pipeline 구성시 Credential 관리 주의 사항

1. 관련 개념 정리:

    CI/CD 파이프라인에서 자격 증명(비밀번호, API 키, 토큰 등)은 절대 코드나 설정 파일에 하드코딩하면 안됨
    Azure Key Vault를 사용하여 중앙 집중식으로 비밀 관리
    Service Principal이나 Managed Identity 사용 권장
    GitHub Actions에서는 Secrets 기능 활용

2. 공식문서 링크:- https://learn.microsoft.com/en-us/azure/devops/pipelines/release/azure-key-vault

    https://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/service-principal-managed-identity

3. 정답 키워드:

    Azure Key Vault
    Service Principal / Managed Identity
    Workload Identity Federation
    Get/List 권한
    절대 하드코딩 금지

14. Github Action의 기본구성 이해와 기본 Task명

1. 관련 개념 정리:

    GitHub Actions는 workflow 파일(.github/workflows/)로 정의
    주요 구성요소: Workflows, Events, Jobs, Steps, Actions, Runners
    기본 Task: checkout, setup-node, cache, upload-artifact, deploy 등

2. 공식문서 링크:- https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions

    https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions

3. 정답 키워드:

    actions/checkout@v4
    actions/setup-node
    actions/upload-artifact / download-artifact
    actions/cache
    on: [push, pull_request, workflow_dispatch]
    jobs / steps / runs-on / uses / with

15. Storage의 암호화 방식

1. 관련 개념 정리:

    Azure Storage는 기본적으로 Server-side Encryption (SSE) 제공
    암호화 방식: Microsoft-managed keys, Customer-managed keys (CMK), Customer-provided keys
    256-bit AES 암호화 사용
    DB 컬럼 암호화는 Storage 암호화가 아님 (Application-level 암호화)

2. 공식문서 링크:- https://learn.microsoft.com/en-us/azure/storage/common/storage-service-encryption

    https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption
    https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-overview

3. 정답 키워드:

    Server-side Encryption (SSE)
    256-bit AES 암호화
    Microsoft-managed keys (기본값)
    Customer-managed keys (CMK)
    항상 활성화 (비활성화 불가)

16. Azure Native 보안 서비스

1. 관련 개념 정리:

    Microsoft Defender for Cloud (구 Azure Security Center)
    Azure Key Vault
    Azure DDoS Protection
    Azure Firewall
    Azure Web Application Firewall (WAF)
    Microsoft Sentinel
    Azure Policy

2. 공식문서 링크:- https://learn.microsoft.com/en-us/azure/sentinel/overview

    https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction

3. 정답 키워드:

    Microsoft Defender for Cloud
    Microsoft Sentinel (SIEM/SOAR)
    Azure Key Vault
    Azure Firewall
    Azure DDoS Protection
    Azure Web Application Firewall (WAF)
    Azure Policy

17. Azure Monitoring 서비스 활용시 비용적인 측면의 Log Data 관리

1. 관련 개념 정리:

    Azure Monitor Log Analytics의 데이터 보존 기간 조정 (기본 31일)
    데이터 아카이빙 (Archive tier 활용)
    로그 샘플링 구성
    불필요한 로그 수집 중단
    Commitment Tier 활용으로 비용 절감

2. 공식문서 링크:- https://learn.microsoft.com/en-us/azure/azure-monitor/logs/cost-logs

    https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-retention-configure
    https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/best-practices-cost

3. 정답 키워드:

    Data Retention 기간 조정 (기본 31일)
    Archive/Long-term retention 활용
    Commitment Tier (일정 용량 약정시 할인)
    Basic Logs (쿼리 제한, 저비용)
    불필요한 로그 필터링/샘플링

18. Migration From VMWare VM: 제약조건, 구성법

1. 관련 개념 정리:

    Azure Migrate 도구 사용
    VMware vSphere 6.0 이상 필요
    네트워크 요구사항: 443 포트 개방
    지원되는 OS 및 디스크 크기 제한
    Azure Site Recovery를 통한 마이그레이션 옵션

2. 공식문서 링크:- https://learn.microsoft.com/en-us/azure/migrate/vmware/migrate-support-matrix-vmware-migration

    https://learn.microsoft.com/en-us/azure/migrate/vmware/migrate-support-matrix-vmware

3. 정답 키워드:

    vCenter Server 6.5 이상
    ESXi 6.5 이상
    443 포트 개방 필요
    Azure Migrate appliance 배포
    디스크 크기 제한 (최대 64TB)
    VMFS Datastore 지원

19. PV/PVC with Azure File 구성시 필요한 Network 설정

1. 관련 개념 정리:

    Private Endpoint 설정 필요
    Storage Account의 네트워크 보안 설정
    AKS 클러스터와 Storage Account 간 네트워크 연결
    서비스 엔드포인트 또는 Private Endpoint 사용

2. 공식문서 링크:- https://learn.microsoft.com/en-us/azure/aks/azure-csi-files-storage-provision

    https://learn.microsoft.com/en-us/azure/storage/common/storage-private-endpoints

3. 정답 키워드:

    Private Endpoint 구성
    networkEndpointType: privateEndpoint (StorageClass)
    privatelink.file.core.windows.net (Private DNS Zone)
    Virtual Network Link 설정
    445 포트 (SMB/CIFS)

20. Data Platform 구축시 요구사항정의와 그에 따른 Cloud특징

1. 관련 개념 정리:

    확장성(Scalability)
    고가용성(High Availability)
    탄력성(Elasticity)
    보안성(Security)
    비용 효율성(Cost Efficiency)

2. 공식문서 링크: 이 문제는 문제 읽기 능력을 테스트하는 것으로 보이므로, 일반적인 클라우드 데이터 플랫폼 특징을 정리합니다.

3. 정답 키워드:

    Scalability (무제한 확장)
    Pay-as-you-go (사용한 만큼 지불)
    Managed Service (관리형 서비스)
    Global Distribution (글로벌 배포)
    Built-in Security (기본 보안)

21. 구독 할당 및 관리에 대한 이해

1. 관련 개념 정리:

    Azure RBAC (Role-Based Access Control)
    Owner, Contributor, Reader 역할
    구독 레벨 권한 관리
    Management Group을 통한 다중 구독 관리

2. 공식문서 링크:- https://learn.microsoft.com/en-us/azure/role-based-access-control/overview

    https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles
    https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-steps

3. 정답 키워드:

    Owner (모든 권한 + 권한 부여)
    Contributor (모든 권한 - 권한 부여 제외)
    Reader (읽기 전용)
    User Access Administrator (권한 관리 전용)
    역할 할당 범위 (Management Group/Subscription/Resource Group/Resource)

22. Vnet Flow log 서비스 종류와 특징

1. 관련 개념 정리:

    기존: NSG Flow Logs
    신규: VNet Flow Logs (2023년 출시)
    VNet Flow Logs는 NSG Flow Logs보다 향상된 기능 제공
    Virtual Network 전체에 대한 가시성 제공

2. 공식문서 링크:- https://learn.microsoft.com/en-us/azure/network-watcher/vnet-flow-logs-overview

    https://learn.microsoft.com/en-us/azure/network-watcher/nsg-flow-logs-overview

3. 정답 키워드:

    VNet Flow Logs (신규, 2023년 출시)
    NSG Flow Logs (기존, 2027년 retirement 예정)
    VNet Flow Logs 장점: Virtual Network 전체 가시성, Azure Virtual Network Manager 지원
    암호화 상태 평가 지원
    단일 로깅 포인트 (NSG는 여러 레벨 필요)
