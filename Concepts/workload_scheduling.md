# Kubernetes 아키텍처와 동작 흐름

## 개요

Kubernetes(K8s)는 컨테이너화된 애플리케이션의 배포, 확장, 복구를 자동화하는 오케스트레이션 플랫폼.

**선언형(Declarative) 모델**로 동작.

- 사용자는 "원하는 상태(Desired State)"를 YAML로 정의해 등록
- 시스템은 실제 상태(Current State)를 원하는 상태에 맞추는 작업을 계속 반복 (Reconciliation Loop)

```
사용자: "app Pod 2개 실행" 등록
   ↓
K8s: 현재 0개 → 2개 생성
   ↓
Pod 1개 장애 발생 → 현재 1개 → 1개 추가 생성 (자동 복구)
```

---

## 1. 클러스터 구성

클러스터는 두 종류의 노드(VM 또는 물리 서버)로 구성.

| 노드                            | 역할                                              |
| ------------------------------- | ------------------------------------------------- |
| **Control Plane (Master Node)** | 클러스터 관리. 상태 저장, 스케줄링, 컨트롤러 실행 |
| **Worker Node**                 | 사용자 애플리케이션 컨테이너 실행                 |

### 1.1 Control Plane 컴포넌트

| 컴포넌트                  | 역할                                                                                                |
| ------------------------- | --------------------------------------------------------------------------------------------------- |
| `kube-apiserver`          | 클러스터의 유일한 진입점. 모든 요청의 인증, 인가, 검증 처리. 컴포넌트 간 통신은 전부 apiserver 경유 |
| `etcd`                    | 분산 키-값 저장소. 모든 리소스 정의와 상태 저장. apiserver만 직접 접근                              |
| `kube-controller-manager` | 컨트롤러 집합 실행. 리소스를 watch하다가 원하는 상태와 차이 발생 시 보정                            |
| `kube-scheduler`          | 노드 미배정 Pod 감지. 리소스 요구량, 제약 조건 기준으로 실행 노드 결정                              |

### 1.2 노드 공통 컴포넌트

| 컴포넌트            | 역할                                                                                                                  |
| ------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `kubelet`           | 모든 노드에서 실행되는 에이전트. 자기 노드에 배정된 Pod를 apiserver에서 받아 container runtime으로 실행하고 상태 보고 |
| `container runtime` | 실제 컨테이너 실행 엔진 (예: containerd)                                                                              |
| `kube-proxy`        | Service 정보를 watch해 노드에 네트워크 규칙(iptables/IPVS) 설정. Service로 들어온 트래픽을 Pod로 전달                 |

### 1.3 클라이언트 / 설치 도구

| 도구      | 역할                                                                      |
| --------- | ------------------------------------------------------------------------- |
| `kubectl` | apiserver에 요청을 보내는 CLI                                             |
| `kubeadm` | 클러스터 설치 도구. `init`으로 Control Plane 구성, `join`으로 Worker 등록 |

### 1.4 통신 원칙

- 모든 컴포넌트는 **apiserver하고만 통신**
- 각 컴포넌트는 apiserver를 **watch**하다가 자기 담당 리소스 변경 시 동작
- 상태 변경은 항상 apiserver → etcd 순서로 기록

```
           kubectl
              │
              ▼
etcd ◀── kube-apiserver ◀── controller-manager / scheduler (watch)
              ▲
              │ (watch / 상태 보고)
     kubelet, kube-proxy (각 노드)
```

---

## 2. 클러스터 부트스트랩

```bash
# Control Plane 노드
kubeadm init

# Worker 노드
kubeadm join <apiserver-주소>:6443 --token <token> --discovery-token-ca-cert-hash <hash>
```

1. `kubeadm init` 실행 시 Control Plane 노드의 kubelet과 container runtime 위에 apiserver, etcd, scheduler, controller-manager가 **Static Pod**로 기동
   - 따라서 Control Plane 노드에도 kubelet과 container runtime 필요
2. `kubeadm join` 실행 시 Worker의 kubelet이 apiserver에 노드로 등록
3. 이후 Worker kubelet은 apiserver와 연결을 유지하며 Pod 실행 지시 수신

---

## 3. 리소스 개요

리소스는 두 분류로 구분.

- **Controller 리소스**: 다른 리소스를 생성하거나 제어
- **Object 리소스**: 단독으로 하나의 인프라 기능 수행

| 분류       | 리소스                    | 역할                                                    |
| ---------- | ------------------------- | ------------------------------------------------------- |
| Controller | `Deployment`              | Pod 배포, 업데이트 전략 관리 (RollingUpdate / Recreate) |
| Controller | `ReplicaSet`              | 지정한 수의 Pod 복제본 유지                             |
| Controller | `HorizontalPodAutoscaler` | 메트릭 기반 replicas 자동 조절                          |
| Object     | `Pod`                     | 컨테이너 실행 최소 단위                                 |
| Object     | `Service`                 | Pod 집합에 고정 접근 지점과 로드밸런싱 제공             |
| Object     | `ConfigMap`               | 일반 설정값 (환경변수, 설정파일)                        |
| Object     | `Secret`                  | 민감정보 (토큰, 인증서, 비밀번호)                       |
| Object     | `PersistentVolumeClaim`   | 저장소 요청                                             |
| Object     | `PersistentVolume`        | 실제 저장소                                             |
| 그룹       | `Namespace`               | 리소스의 논리적 그룹. 이름 충돌 방지, 권한 분리         |

### 3.1 리소스 간 참조 관계

리소스는 **label/selector** 또는 **이름 참조**로 연결.

```
HPA ──scaleTargetRef──▶ Deployment ──생성──▶ ReplicaSet ──생성──▶ Pod
                                                                   ▲
Service ──selector (name: app)─────────────────────────────────────┘
ConfigMap, Secret ◀──envFrom── Pod
Pod ──volumes──▶ PVC ──바인딩──▶ PV ──hostPath──▶ 노드 디스크
```

---

## 4. 동작 흐름

### 4.1 Pod 배포 흐름

`kubectl apply -f deployment.yaml` 실행 시:

| 단계 | 주체                | 동작                                                                                         |
| ---- | ------------------- | -------------------------------------------------------------------------------------------- |
| 1    | kubectl → apiserver | 요청 전송. 인증, 인가, 스키마 검증                                                           |
| 2    | apiserver → etcd    | Deployment 저장. 이 시점엔 기록만                                                            |
| 3    | controller-manager  | Deployment 컨트롤러가 ReplicaSet 생성                                                        |
| 4    | controller-manager  | ReplicaSet 컨트롤러가 replicas 수만큼 Pod 객체 생성 (`nodeName` 비어 있음)                   |
| 5    | scheduler           | 미배정 Pod 감지 → 노드 필터링, 점수 계산 → 노드 바인딩                                       |
| 6    | kubelet (Worker)    | 자기 노드 배정 Pod 감지 → ConfigMap/Secret 주입, 볼륨 마운트 → container runtime에 실행 지시 |
| 7    | kubelet → apiserver | Pod 상태 보고 (`Pending` → `Running`) → etcd 반영                                            |

각 단계 결과는 apiserver를 거쳐 etcd에 기록. 다음 단계 컴포넌트가 그 변화를 watch해서 이어받는 구조.

### 4.2 트래픽 흐름

1. Service 생성 시 selector와 일치하는 Pod IP 목록이 **EndpointSlice**로 자동 관리
2. 각 노드의 kube-proxy가 Service와 EndpointSlice를 watch해 iptables/IPVS 규칙 설정
3. 클라이언트 요청이 `Service:80`으로 유입 → 커널 규칙에 따라 Pod 하나의 `:8080`으로 전달
4. Pod 추가/삭제 시 EndpointSlice와 규칙 자동 갱신

> kube-proxy는 규칙만 설정. 패킷이 kube-proxy 프로세스를 직접 거치지는 않음 (iptables/IPVS 모드 기준).

### 4.3 스토리지 흐름

1. 관리자가 PV 생성 (예: 노드 `/mnt/data`, 250Mi, `ReadWriteOnce`)
2. PVC 생성 시 용량, accessModes, selector 조건이 맞는 PV와 바인딩
3. Pod가 PVC를 volume으로 참조 → kubelet이 컨테이너 경로(`/var/app`)에 마운트
4. 컨테이너가 쓴 데이터는 PV에 저장 → Pod 재생성 후에도 유지

### 4.4 설정 주입 흐름

1. ConfigMap / Secret 먼저 생성
2. Pod 기동 시 kubelet이 apiserver에서 값을 조회해 환경변수(`envFrom`) 또는 파일(volume)로 주입
3. 환경변수 방식은 값 변경이 실행 중 Pod에 미반영 → Pod 재시작 필요

---

## 5. 운영 중 자동 동작

### 5.1 자가 복구

```
Pod 장애 → ReplicaSet: 실제 1개 < 원하는 2개 → Pod 생성 → scheduler → kubelet 실행
```

### 5.2 오토스케일링

- HPA가 metrics-server에서 CPU 사용률 조회
- 평균 50% 초과 시 Deployment `replicas` 증가 (최대 4), 낮아지면 감소 (최소 2)
- 이후 흐름은 4.1의 3~7단계와 동일

### 5.3 롤링 업데이트

- 이미지 변경 시 Deployment가 새 ReplicaSet 생성
- `RollingUpdate`: 새 Pod 증가, 기존 Pod 감소를 반복하며 점진적 교체. 무중단
- `Recreate`: 기존 Pod 전부 종료 후 새 Pod 생성. 다운타임 발생
- 기존 ReplicaSet은 롤백용으로 보존
