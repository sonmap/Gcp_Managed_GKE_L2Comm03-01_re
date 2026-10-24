# 2026-10-24 L2Comm03-01_re 수행 결과

[실행 절차 RUNBOOK](RUNBOOK.md) | 대상 저장소: sonmap/Gcp_Managed_GKE_L2Comm03-01_re

이 문서는 작업 증거와 결과를 정리한 기록이다. 명령 실행 방법은 RUNBOOK을 따른다.
클라우드에 다시 접속하여 검증한 결과가 아니라 첨부 로그와 사용자 제공 실행 출력을 기준으로 작성했다.
기록의 UTC 시간과 한국 시간은 구분한다. 시각이 없는 항목은 실행 시각을 추정하지 않는다.

## 1. 자료 범위와 판정 기준

| 자료 | 확인 내용 | 증거 범위 |
|---|---|---|
| Pasted text(20261024-130005).txt | 952행, admin 활성 계정, infra-son01 터미널 | 아래 A 항목의 원본 |
| Pasted text (2).txt | 첫 번째 파일과 바이트 단위 동일 | 별도 사용자 실행 증거로 중복 계산하지 않음 |
| Pasted text (3).txt | 첫 번째 파일과 바이트 단위 동일 | 별도 사용자 실행 증거로 중복 계산하지 않음 |
| 같은 대화의 실행 출력 | GKE·Pod, 빌드 성공 및 실패 로그 | 아래 B 항목. 첨부 파일과 구분 |

첨부 3개 파일 SHA-256:
01f91dd3062917e67fe14d883aaf88f1135f0a7c19c65d75084e081d83a5b195

A = 첨부 로그에서 결과 확인 / B = 대화의 사용자 출력에서 확인 / C = 지시·계획만 존재하거나 완료 출력 없음.
터미널의 Linux 사용자명과 gcloud 활성 계정은 다르다. 작업 계정은 auth 출력과 IAM 결과를 기준으로 구분했다.
원본 로그에는 로그인 인증 정보와 .oefj 개인키가 포함되어 있어 원문은 저장소에 복제하지 않았다.
아래에는 키 내용·키 ID·OAuth 인증값을 기록하지 않는다.

## 2. 프로젝트 및 실측 네트워크

| 역할 | 프로젝트 | 프로젝트 번호 | 수행자 |
|---|---|---:|---|
| Host | gcp-prod-edp-hub-vpchost | 620081195575 | admin@sonmap.net |
| Edge / AR / central02 | gcp-sbx-edp-gke01 | 869916265282 | user01@sonmap.net |
| SVC / GKE | gcp-sbx-edp-comn-509423 | 1003210826985 | 관리·IAM: admin / GKE 생성: cho |
| 생성 VM | gcp-prod-edp-edge-509423 | 541022739403 | cho@sonmap.net |

| 항목 | 결과 | 판정·근거 |
|---|---|---|
| Shared VPC Host | vpc-prod-edp-hub / gcp-prod-edp-hub-vpchost | A: 세 서비스 프로젝트 get-host-project 결과 일치, 원본 123~140행 부근 |
| Node Primary | subnet-prod-l2comn-gke / 10.253.0.0/28 | A: create 및 describe, 165~190행 |
| Pod Secondary | subnet-prod-l2comn-gke-pod / 10.253.4.0/22 | A: describe 184~186행 |
| Private Endpoint 서브넷 | subnet-prod-l2comn-gke-cp / 10.253.2.0/28 | A: create·describe 168~204행 |
| Node 서브넷 Private Google Access | true | A: describe 180행 부근 |
| CP 서브넷 Private Google Access | false | A: describe 198행 부근. Node 서브넷 설정과 구분 |
| 관리 VM 서브넷 | subnet-prod-edp-infra-admin-an3 / 172.32.10.0/24 | A: subnet list |
| l2comn-dev / central02 | 172.32.10.3 / 172.32.10.4 | B 및 기존 구성. 첨부의 VM describe 증거는 없음 |

초기 subnet list에는 Node 대역이 10.253.0.0/24로 표시된 부분이 있으나,
후속 생성·상세 조회에는 /28로 확인된다. 이 기록은 후속 describe의 /28을 기준으로 한다.
삭제·재생성 등 중간 작업은 로그에 없으므로 변경 경위를 추정하지 않는다.
현재 운영값은 RUNBOOK의 describe 명령으로 재확인한다.

## 3. admin@sonmap.net 수행 결과 — 첨부 로그 A

실행 터미널: infra-son01. 시작 auth list에서 admin 계정 ACTIVE 확인.

| 순서 | RUNBOOK | 적용 위치 | 수행 내용 | 결과·원본 근거 |
|---|---|---|---|---|
| 1 | 0 | 네 프로젝트 | 프로젝트 번호 조회 | 성공: 51/116행 |
| 2 | 1 | Shared VPC | SVC·Edge·생성 VM 프로젝트의 Host 조회 | Host 일치 |
| 3 | 1 | Host | SVC·Edge associated-projects add | Updated 출력. 기존 연결과 동일 Host |
| 4 | 2 | Host | Node·CP 서브넷 생성 및 Pod Secondary 설정 | Created/describe 확인: 165~204행 |
| 5 | 3-A | Host | GKE 서비스 ID 준비 | service-620081195575@container-engine-robot.iam.gserviceaccount.com: 207행 |
| 6 | 3-B | Host | SVC GKE Agent의 hostServiceAgentUser | 정책 반영: 429~478행 |
| 7 | 3-B | Node·CP 서브넷 | GKE Agent, Google APIs Agent, cho에 networkUser | 두 서브넷 정책 반영: 479~526행 |
| 8 | 3-C | SVC | sa-l2comm-node 생성 | Created: 599~600행 |
| 9 | 3-C | SVC | Node SA의 defaultNodeServiceAccount | 정책 반영: 601행 이후 |
| 10 | 3-C | SVC | cho의 container.admin / serviceUsageConsumer | 정책 반영: 656~770행 |
| 11 | 3-C | Node SA | cho의 iam.serviceAccountUser | 정책 반영: 771~777행 |
| 12 | 3-D | Host | SVC GKE Agent의 compute.securityAdmin | 정책 반영: 783행 이후 |
| 13 | 7 | Host | allow-l2comm-nfs-from-gke 생성 | TCP2049 허용 생성 성공: 842~845행 |
| 14 | 7 | Host | allow-iap-ssh-central02 재생성 시도 | already exists: 847~848행. 신규 생성 성공으로 기록하지 않음 |

### 확인된 핵심 IAM 관계

| 권한 받는 주체 | 적용 위치 | 역할 | 증거 |
|---|---|---|---|
| service-1003210826985@container-engine-robot.iam.gserviceaccount.com | Host | roles/container.hostServiceAgentUser | A |
| 위 GKE Agent | Host | roles/compute.securityAdmin | A |
| 위 GKE Agent | Node·CP 서브넷 각각 | roles/compute.networkUser | A |
| 1003210826985@cloudservices.gserviceaccount.com | Node·CP 서브넷 각각 | roles/compute.networkUser | A |
| cho@sonmap.net | Node·CP 서브넷 각각 | roles/compute.networkUser | A |
| sa-l2comm-node@gcp-sbx-edp-comn-509423.iam.gserviceaccount.com | SVC | roles/container.defaultNodeServiceAccount | A |
| cho@sonmap.net | SVC | roles/container.admin | A |
| cho@sonmap.net | SVC | roles/serviceusage.serviceUsageConsumer | A |
| cho@sonmap.net | Node SA | roles/iam.serviceAccountUser | A |
| user01@sonmap.net | SVC | roles/container.admin | A: IAM 스냅샷에서 존재 확인. 부여 명령은 첨부에서 확인 안 됨 |
| admin@sonmap.net | SVC | roles/owner | A: 프로젝트 IAM 스냅샷 |

cho는 첨부의 SVC 프로젝트 Owner 목록에 없다. 실제 권한 오류와 일치하므로
SA 생성·IAM 변경은 admin이 수행하고 cho에게 위임 권한을 부여하는 절차를 유지한다.
SVC GKE 서비스 에이전트의 존재는 IAM 출력으로 확인되나, SVC 서비스 ID 생성 명령의 성공 출력은 첨부에 없다.

## 4. cho@sonmap.net — GKE 생성·접속 결과 B

| 작업 | 확인된 결과 | RUNBOOK |
|---|---|---|
| SVC Node SA 생성·IAM 변경 시도 | iam.serviceAccounts.create / setIamPolicy 거부. admin 처리로 전환 | 3-C |
| 최초 Private GKE 생성 | master authorized networks 활성화 필요 오류 | 4 |
| 명령 보완 | enable-master-authorized-networks 추가, private endpoint 승인 IP 제한 미적용 유지 | 4 |
| 클러스터 가동 | Pod가 gk3-gkea-prod-l2comn-bat 노드에 배치되어 실행됨. 인프라 가동 확인 | 4/10 |
| get-credentials | kubeconfig entry generated for gkea-prod-l2comn-batch | 8 |
| kubectl·플러그인 | command not found / Unable to locate package 발생 | 사전 준비 B |
| 후속 kubectl 사용 | describe/logs가 서버 결과를 반환함. 도구 사용 가능 상태 확인 | 사전 준비 B |

클러스터 create 완료 원문은 첨부에 없으므로 create의 RUNNING 요약을 임의로 만들지 않는다.
Pod에서 확인된 값: Node IP 10.253.0.2, Pod IP 10.253.4.13, Zone asia-northeast3-c.
배치 Pod 요청·제한은 4 CPU / 16Gi, ephemeral-storage 1Gi였다.
Node 전체 CPU·메모리·MAX_PODS·노드별 Pod CIDR·전체 Node 수는 이 자료만으로 확정할 수 없다.

## 5. user01@sonmap.net — Edge AR 빌드 결과 B

| 항목 | 결과 |
|---|---|
| AR 저장소 | gcp-sbx-edp-gke01 / asia-northeast3 / ar-l2comm-python |
| 최초 조회 | Listed 0 items |
| 최초 Pod 오류 | python-bq-batch:v1: not found |
| cho로 빌드 시도 | gcp-sbx-edp-gke01_cloudbuild 버킷 접근 거부 |
| 정정된 수행 계정 | Edge Owner user01@sonmap.net |
| 재실행 결과 | Cloud Build SUCCESS. 성공 출력 자체에는 제출자 이메일이 없으며 계정 전환 안내 후 보고된 결과 |
| Build ID | 9b5b0c93-d8c6-4240-a78b-d72e887b916a |
| 생성 시각 | 2026-10-24 12:33:03 UTC / 21:33:03 KST |
| 빌드 소요 | 41초 |
| 이미지 | asia-northeast3-docker.pkg.dev/gcp-sbx-edp-gke01/ar-l2comm-python/python-bq-batch:v1 |
| Pod에서 확인된 digest | sha256:69b1d1c3c3847e8cde3aab863d3da55854cfdd4fa07ece846c6414d698fdde0c |

빌드 성공 및 Pod의 Image ID/Container ID 존재로 이미지 등록·다운로드·컨테이너 시작을 확인했다.
AR Reader/Writer 정책의 실제 출력과 Build SA 생성 성공 출력은 첨부에 없으므로 각각의 적용 완료를 따로 단정하지 않는다.
빌드 제출 사용자와 cloudbuild.yaml의 Build SA, GKE Node SA, .oefj 소유 SA는 서로 다르다.

## 6. NFS·.oefj·Pod 실행 결과

| 항목 | 판정 | 근거 및 해석 |
|---|---|---|
| NFS TCP2049 방화벽 | A: 생성 성공 | source는 당시 NODE_CIDR /28, target l2comm-nfs로 명령 실행 |
| central02 NFS 설정 및 VM 태그 | C: 개별 완료 출력 없음 | RUNBOOK 7절에서 재확인 |
| Pod 이미지 Pull | B: 해결 | Image ID와 Container ID 존재, 컨테이너 시작 |
| 실행 실패 | B: Failed / Exit Code 2 | Missing or unreadable SA key: /home/jupyter/.oefj |
| 키 복사 | B: 사용자 완료 보고 | central02의 실제 목적지·UID별 가독성 출력은 아직 없음 |
| 기존 원본 키 위치 | A: 구 저장소 rsvp/.oefj 존재 | infra-son01의 Gcp_Managed_GKE_L2Comm03-01 경로. central02 배치 증거와 구분 |
| Pod 삭제·재실행 | C: 실행 방법 안내 | 완료 출력 없음 |
| BigQuery 테스트 성공 | C: 아직 확인 안 됨 | L2COMM SUCCESS 등 최종 로그 없음 |

키 소유 SA: sa-l2comm-bq-test@pjt-c-admin.iam.gserviceaccount.com.
기록에는 키 내용 대신 인증 주체·경로만 남긴다.
현재 확정된 최종 상태는 이미지 문제 해결 후 .oefj 읽기 실패까지다. 배치 전체 성공으로 기록하지 않는다.

| 경로 역할 | 올바른 위치 |
|---|---|
| central02 Git 소스 내부 키 | ~/Gcp_Managed_GKE_L2Comm03-01_re/mnt/batch_gke/l2/rsvp/.oefj |
| central02 NFS bind 경로 | /srv/l2comm-nfs/batch_gke/l2/rsvp/.oefj |
| Pod 키 경로 | /home/jupyter/.oefj |

## 7. 오류와 최종 적용 방향

| 현상 | 최종 적용 방향 | RUNBOOK |
|---|---|---|
| cho의 SA 생성·IAM 변경 거부 | admin이 관리 작업, cho는 위임된 GKE 생성 권한 사용 | 3-A/3-C/8 |
| Private Endpoint 생성 400 | enable-master-authorized-networks 추가 | 4 |
| kubectl·플러그인 미설치 | Google Cloud APT 저장소 등록 후 두 패키지 설치 | 사전 준비 B |
| Edge AR 이미지 없음 | user01로 Edge 프로젝트 Cloud Build 실행 | 5/5-A |
| cho의 빌드 staging bucket 접근 거부 | 수행 역할에 맞춰 user01로 전환 | 5 |
| .oefj 읽기 실패 | central02 경로·권한 확인 후 실패한 Pod만 삭제·재실행 | 7-A/9-A |
| SSH 방화벽 already exists | 기존 규칙 describe로 적합성 확인. 무조건 삭제·재생성하지 않음 | 7 |
| 로그인 중 Ctrl+C·KeyboardInterrupt | 중단 자체를 권한 적용 성공으로 계산하지 않음. 후속 Updated 출력으로 판정 | 3-B |

## 8. 후속 확인 명령 — 조회용, 자동 실행 아님

각 명령은 해당 권한이 있는 계정에서 수행한다. 아래 결과를 얻으면 이 문서의 C 항목을 보완할 수 있다.

### Host 관리자

```bash
gcloud compute networks subnets describe subnet-prod-l2comn-gke \
  --project=gcp-prod-edp-hub-vpchost --region=asia-northeast3 \
  --format='yaml(ipCidrRange,secondaryIpRanges,privateIpGoogleAccess)'

gcloud compute firewall-rules describe allow-iap-ssh-central02 \
  --project=gcp-prod-edp-hub-vpchost

gcloud compute firewall-rules describe allow-l2comm-nfs-from-gke \
  --project=gcp-prod-edp-hub-vpchost
```

### Edge Owner — 빌드·이미지・VM

```bash
gcloud builds describe 9b5b0c93-d8c6-4240-a78b-d72e887b916a \
  --project=gcp-sbx-edp-gke01 --region=asia-northeast3

gcloud artifacts docker images list \
  asia-northeast3-docker.pkg.dev/gcp-sbx-edp-gke01/ar-l2comm-python \
  --project=gcp-sbx-edp-gke01 --include-tags

gcloud compute instances describe central02 \
  --project=gcp-sbx-edp-gke01 --zone=asia-northeast3-a \
  --format='yaml(networkInterfaces,tags)'
```

### central02 내부 — 내용 출력 없이 키·마운트 확인

```bash
findmnt /srv/l2comm-nfs
sudo exportfs -v
ls -l /srv/l2comm-nfs/batch_gke/l2/rsvp/.oefj
```

### GKE 접속 가능 계정 — Node 규모 및 실제 배치 결과

```bash
kubectl get nodes \
  -o custom-columns='NODE:.metadata.name,CPU:.status.capacity.cpu,MEMORY:.status.capacity.memory,MAX_PODS:.status.allocatable.pods,POD_CIDR:.spec.podCIDR'
kubectl get pods -n l2comm-batch -o wide
```

재실행은 RUNBOOK 9-A를 따른다. 기존 client_gke는 성공하면 로그 출력 후 Pod를 삭제하므로
성공 로그를 별도로 보관하여 BQ 인증·업무 완료를 증빙한다.
