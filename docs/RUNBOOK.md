# L2Comm03-01_re 실행 절차

인프라는 아래 gcloud 명령을 단계별로 실행한다. 인프라 생성용 shell 파일을 실행하지 않는다.
NFS 설정만 setup-nfs-server.sh를 사용한다. 기존 run-gke.sh / client_gke의 업무 실행은 유지한다.
명령은 실제 클라우드에 적용하지 않은 절차이며 각 단계 출력 확인 후 다음 단계로 진행한다.
기존 리소스가 있으면 describe로 확인하고 create를 생략한다. 실패를 무시하고 진행하지 않는다.

## 수행 Owner와 프로젝트

| 적용 프로젝트 | 수행 관리자·사용자 | 수행 작업 |
|---|---|---|
| HOST_PROJECT: gcp-prod-edp-hub-vpchost | admin@sonmap.net | Host API/서비스 ID, 서브넷, Host IAM, 방화벽 |
| EDGE_PROJECT: gcp-sbx-edp-gke01 | user01@sonmap.net | Edge API, AR/Build SA/IAM, central02 태그 |
| SVC_PROJECT: gcp-sbx-edp-comn-509423 | admin@sonmap.net: 관리자 / cho@sonmap.net: GKE 생성 사용자 | admin: API/서비스 ID, Node SA/IAM, 사용자 권한 부여 / cho: GKE 생성 |
| CREATE_VM_PROJECT: gcp-prod-edp-edge-509423 | cho@sonmap.net | l2comn-dev 관리. GKE 생성 대상은 SVC_PROJECT |

admin@sonmap.net은 전체 관리자이며 SVC의 API·서비스 ID·SA 생성·IAM 설정을 담당한다.
cho@sonmap.net은 SVC Owner가 아님을 확인했다. admin이 위임한 권한으로 GKE 생성만 수행한다.
VM 위치가 권한 적용 프로젝트를 결정하지 않는다. 명령의 --project와 실제 인증 계정이 기준이다.
Shared VPC 연결 변경은 프로젝트 Owner만으로 충분하지 않으며 별도의 Shared VPC Admin 권한이 필요하다.
각 계정은 자기 프로젝트 번호를 조회해 다른 담당자에게 공유한다. 다른 프로젝트 describe 권한이 없다면
0절의 네 프로젝트 번호 일괄 조회를 강행하지 말고 공유받은 번호를 export한다.
gcloud auth login 이후 ACTIVE 계정을 확인한다. 이미 인증되어 있으면 gcloud config set account EMAIL로 전환할 수 있다.

## 사전 준비. OS와 gcloud만 있는 두 VM의 도구 설치

l2comn-dev와 central02 각각 SSH 접속 후 실행한다. gcloud는 이미 설치되어 있으므로 재설치하지 않는다.
아래는 VM 내부 패키지 설치 명령이며 별도 설치 sh 파일을 만들지 않는다.

| 도구 | l2comn-dev | central02 | 용도 |
|---|---|---|---|
| git | 설치 | 설치 | 동일 저장소 clone/pull |
| python3 | 설치 | 설치 | client_gke 보조 처리와 점검 |
| python3-venv, python3-pip | 설치 | 설치 | 필요 시 분리된 로컬 Python 환경 |
| kubectl | 설치 | 설치 | 클러스터 확인, central02의 Pod 생성 |
| gke-gcloud-auth-plugin | 설치 | 설치 | GKE 인증 |
| nfs-kernel-server | 설치 안 함 | NFS sh에서 설치 | NFS 서버 |
| Docker / Helm | 필수 아님 | 필수 아님 | 빌드는 Cloud Build, 실행은 기존 client_gke |

### A. 공통 패키지 — 두 VM 각각

```bash
sudo apt-get update
sudo apt-get install -y git python3 python3-venv python3-pip ca-certificates curl gnupg
git --version
python3 --version
python3 -m pip --version
gcloud version
```

### B. kubectl·GKE 인증 플러그인 설치 — 두 VM 각각

l2comn-dev와 central02에서 각각 수행한다. gcloud가 설치되어 있어도 kubectl과 인증 플러그인은 별도 설치가 필요하다.
이 단계의 설치 확인이 성공하기 전에는 kubectl get nodes를 반복하지 않는다.

#### B-1. 설치 실패 증상과 원인

| 증상 | 의미 | 조치 |
|---|---|---|
| kubectl: command not found | 실행 파일 미설치 또는 PATH 미등록 | 아래 설치 후 command -v 확인 |
| gke-gcloud-auth-plugin: command not found | GKE 인증 도구 미설치 또는 PATH 미등록 | 아래 설치 후 버전 확인 |
| E: Unable to locate package kubectl | APT 목록에서 패키지를 찾지 못함 | Google Cloud 저장소 등록·갱신 확인 |
| E: Unable to locate package google-cloud-cli-gke-gcloud-auth-plugin | APT 목록에서 플러그인을 찾지 못함 | 동일 |
| kubeconfig entry generated | 접속 설정 파일만 생성됨 | 도구 설치 성공을 의미하지 않음 |

이번 l2comn-dev에서는 apt-get update에 Ubuntu jammy 저장소만 표시되고 Google Cloud 저장소가 표시되지 않았다.
따라서 Google Cloud APT 저장소 등록부터 진행한다. hash -r이나 get-credentials는 패키지를 설치하지 않는다.

#### B-2. 기존 저장소 확인

```bash
sudo rg 'packages.cloud.google.com/apt' /etc/apt/sources.list /etc/apt/sources.list.d
```

rg가 설치되지 않은 기본 Ubuntu에서는 다음 명령을 사용한다.

```bash
sudo grep -R 'packages.cloud.google.com/apt' /etc/apt/sources.list /etc/apt/sources.list.d
```

등록 내역이 없으면 B-3을 수행한다. 이미 다른 파일에 동일 저장소가 있으면 중복 추가하지 않고 기존 서명 설정을 확인한 뒤 B-4로 진행한다.
검색 결과가 없는 경우의 종료 코드 1은 미등록을 의미한다.

#### B-3. 필수 도구·서명 키·Google Cloud 저장소 등록

아래 블록을 순서대로 실행한다. 각 블록이 실패하면 다음으로 진행하지 않는다.

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo mkdir -p /usr/share/keyrings
```

서명 키를 먼저 내려받는다. curl 실패를 놓치지 않도록 다운로드와 키 변환을 분리한다.

```bash
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg \
  -o /tmp/l2comm-google-cloud-apt-key.asc
```

```bash
sudo gpg --dearmor --yes \
  -o /usr/share/keyrings/cloud.google.gpg \
  /tmp/l2comm-google-cloud-apt-key.asc
sudo chmod 0644 /usr/share/keyrings/cloud.google.gpg
```

```bash
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" \
  | sudo tee /etc/apt/sources.list.d/google-cloud-sdk.list
```

#### B-4. 패키지 목록 갱신 및 설치

```bash
sudo apt-get update
```

출력에 packages.cloud.google.com의 cloud-sdk 저장소가 표시되고 서명·다운로드 오류가 없는지 확인한다.
Ubuntu 저장소만 나타나면 B-2/B-3 등록 상태를 확인한다.

```bash
apt-cache policy kubectl google-cloud-cli-gke-gcloud-auth-plugin
```

두 패키지의 Candidate에 설치 가능한 버전이 표시되어야 한다. Candidate: (none)이면 아직 설치하지 말고 저장소 갱신 오류를 확인한다.

```bash
sudo apt-get install -y kubectl google-cloud-cli-gke-gcloud-auth-plugin
```

#### B-5. 설치 확인

```bash
hash -r
command -v kubectl
command -v gke-gcloud-auth-plugin
kubectl version --client
gke-gcloud-auth-plugin --version
```

두 실행 경로와 버전이 모두 출력되어야 한다. 계속 command not found이면 설치 명령의 전체 출력을 확인한다.
GKE 생성 이후에는 8절에서 get-credentials 후 kubectl 접속을 확인한다. 4절 생성 계정 cho도 동일한 도구를 사용한다.

APT 방식으로 gcloud를 설치한 VM에서는 gcloud components install 대신 위 APT 방식을 사용한다.
패키지 다운로드와 git clone에는 기존 NAT/사내 프록시/인터넷 중계 경로가 필요하다.
외부 IP가 없는 VM에 IAP SSH가 된다고 인터넷 다운로드가 가능한 것은 아니다.
curl 또는 apt update 실패 시 오류 출력을 확인하고 기존 승인된 외부 통신 경로를 점검한다.
이 절차는 별도 sh 파일을 만들지 않으며 GCP IAM·클러스터 설정을 변경하지 않는다.

### C. Python 실행 위치

VM의 Python 3는 도구용이다. 실제 BigQuery 배치는 Pod 안에서 실행한다.
google-cloud-bigquery 등 업무 의존성은 기존 app/requirements.txt와 Dockerfile로 이미지에 설치한다.
두 VM에 업무용 pip 패키지를 일괄 설치할 필요는 없다. 로컬 검증이 필요한 경우에만 별도 venv를 사용한다.
NFS sh는 central02에서만 실행하고 l2comn-dev에는 mnt 마운트를 구성하지 않는다.

## 0. 프로젝트와 변수 — 각 작업 터미널에서 정의

| 변수 | 프로젝트 | 역할 |
|---|---|---|
| HOST_PROJECT | gcp-prod-edp-hub-vpchost | Shared VPC, 서브넷, 방화벽 |
| EDGE_PROJECT | gcp-sbx-edp-gke01 | central02, AR, 빌드, NFS |
| SVC_PROJECT | gcp-sbx-edp-comn-509423 | GKE Autopilot |
| CREATE_VM_PROJECT | gcp-prod-edp-edge-509423 | 생성 VM l2comn-dev |

아래 대역은 사전 IP 승인 및 기존 VPC/피어링/VPN/사내망과 중복 확인이 필요하다.
Node /28은 기본 예약 제외 12개 주소만 제공한다. Pod /22 수용량은 노드별 할당량에 따라 달라진다.

```bash
export HOST_PROJECT="gcp-prod-edp-hub-vpchost"
export EDGE_PROJECT="gcp-sbx-edp-gke01"
export SVC_PROJECT="gcp-sbx-edp-comn-509423"
export CREATE_VM_PROJECT="gcp-prod-edp-edge-509423"

export HOST_PROJECT_NUM="$(gcloud projects describe "$HOST_PROJECT" --format='value(projectNumber)')"
export EDGE_PROJECT_NUM="$(gcloud projects describe "$EDGE_PROJECT" --format='value(projectNumber)')"
export SVC_PROJECT_NUM="$(gcloud projects describe "$SVC_PROJECT" --format='value(projectNumber)')"
export CREATE_VM_PROJECT_NUM="$(gcloud projects describe "$CREATE_VM_PROJECT" --format='value(projectNumber)')"

echo "$HOST_PROJECT_NUM $EDGE_PROJECT_NUM $SVC_PROJECT_NUM $CREATE_VM_PROJECT_NUM"

export REGION="asia-northeast3"
export VM_ZONE="asia-northeast3-a"
export NETWORK="vpc-prod-edp-hub"
export NODE_SUBNET="subnet-prod-l2comn-gke"
export POD_RANGE="subnet-prod-l2comn-gke-pod"
export CP_SUBNET="subnet-prod-l2comn-gke-cp"
export NODE_CIDR="10.253.0.0/28"
export POD_CIDR="10.253.4.0/22"
export CP_CIDR="10.253.2.0/28"
export CLUSTER="gkea-prod-l2comn-batch"
export RELEASE_CHANNEL="regular"
export CREATE_VM_ACCOUNT="cho@sonmap.net"
export EXEC_ACCOUNT="user01@sonmap.net"
export ADMIN_ACCOUNT="admin@sonmap.net"
export HOST_OWNER="$ADMIN_ACCOUNT"
export EDGE_OWNER="$EXEC_ACCOUNT"
export SVC_OWNER="$ADMIN_ACCOUNT"  # SVC 관리 작업 담당. cho는 GKE 생성 사용자
export CREATE_VM_OWNER="$CREATE_VM_ACCOUNT"
export NODE_SA="sa-l2comm-node@${SVC_PROJECT}.iam.gserviceaccount.com"
export BUILD_SA="sa-l2comm-build@${EDGE_PROJECT}.iam.gserviceaccount.com"
export REPO_NAME="ar-l2comm-python"
export DOCKER_IMAGE="${REGION}-docker.pkg.dev/${EDGE_PROJECT}/${REPO_NAME}/python-bq-batch:v1"
```

프로젝트 번호가 모두 숫자인지 확인한다. 계정 변수 선언은 인증을 변경하지 않는다.
각 VM에는 gcloud, git, kubectl, gke-gcloud-auth-plugin이 필요하다.
사용자 로그인 방식이며 VM 부착 SA와 다르다. 사내 패키지 배포 절차로 도구를 준비한다.

## 1. 관리자 — API와 Shared VPC 상태

```bash
gcloud auth login "$ADMIN_ACCOUNT"
gcloud auth list --filter=status:ACTIVE
gcloud services enable compute.googleapis.com container.googleapis.com --project="$HOST_PROJECT"

# SVC 관리자: admin@sonmap.net
gcloud auth login "$SVC_OWNER"
gcloud services enable compute.googleapis.com container.googleapis.com iam.googleapis.com --project="$SVC_PROJECT"

# Edge Owner: user01@sonmap.net
gcloud auth login "$EDGE_OWNER"
gcloud services enable compute.googleapis.com artifactregistry.googleapis.com cloudbuild.googleapis.com iam.googleapis.com --project="$EDGE_PROJECT"

# 전체 관리자: Shared VPC 상태 조회
gcloud auth login "$HOST_OWNER"

gcloud compute shared-vpc get-host-project "$SVC_PROJECT"
gcloud compute shared-vpc get-host-project "$EDGE_PROJECT"
gcloud compute shared-vpc get-host-project "$CREATE_VM_PROJECT"
gcloud compute networks subnets list --project="$HOST_PROJECT" --network="$NETWORK"
```

SVC/EDGE/생성 VM 프로젝트가 이미 같은 Host에 연결되었다면 변경하지 않는다.
SVC 또는 EDGE가 미연결인 경우만 Shared VPC 관리자가 다음을 실행한다.
다른 Host에 연결된 프로젝트를 자동으로 분리하지 않는다.

```bash
gcloud compute shared-vpc associated-projects add "$SVC_PROJECT" --host-project="$HOST_PROJECT"
gcloud compute shared-vpc associated-projects add "$EDGE_PROJECT" --host-project="$HOST_PROJECT"
```

## 2. Host Owner admin@sonmap.net — 네트워크

```bash
gcloud auth login "$HOST_OWNER"
gcloud compute networks subnets create "$NODE_SUBNET" \
  --project="$HOST_PROJECT" --region="$REGION" --network="$NETWORK" \
  --range="$NODE_CIDR" --secondary-range="${POD_RANGE}=${POD_CIDR}" \
  --enable-private-ip-google-access

gcloud compute networks subnets create "$CP_SUBNET" \
  --project="$HOST_PROJECT" --region="$REGION" --network="$NETWORK" \
  --range="$CP_CIDR"

gcloud compute networks subnets describe "$NODE_SUBNET" --project="$HOST_PROJECT" --region="$REGION"
gcloud compute networks subnets describe "$CP_SUBNET" --project="$HOST_PROJECT" --region="$REGION"
```

별도 Service Secondary는 생성하지 않고 GKE 관리형 Service 범위를 사용한다.
Private Google Access는 인터넷 연결을 제공하지 않는다. VM의 apt/GitHub 접속은 기존 사내 프록시·NAT 등의 경로가 필요하다.
이 절차는 NAT를 새로 만들지 않는다.

## 3. Owner별 — GKE 서비스 에이전트와 Node SA

수행 순서: 3-A 서비스 ID 준비 → 3-B Host 권한 → 3-C Node SA → 3-D 선택 방화벽 권한.

### 3-A. Host·SVC 서비스 ID 생성 — admin@sonmap.net

```bash
# Host Owner: admin@sonmap.net
gcloud auth login "$HOST_OWNER"
gcloud beta services identity create --service=container.googleapis.com --project="$HOST_PROJECT"

# SVC Owner: cho@sonmap.net
gcloud auth login "$SVC_OWNER"
gcloud beta services identity create --service=container.googleapis.com --project="$SVC_PROJECT"
```

### 3-B. Host 프로젝트·서브넷 권한 — admin@sonmap.net

SVC 서비스 에이전트와 cho 계정에 부여하더라도 적용 리소스가 Host이므로 Host Owner가 실행한다.

```bash
gcloud auth login "$HOST_OWNER"

gcloud projects add-iam-policy-binding "$HOST_PROJECT" \
  --member="serviceAccount:service-${SVC_PROJECT_NUM}@container-engine-robot.iam.gserviceaccount.com" \
  --role=roles/container.hostServiceAgentUser

gcloud compute networks subnets add-iam-policy-binding "$NODE_SUBNET" \
  --project="$HOST_PROJECT" --region="$REGION" \
  --member="serviceAccount:service-${SVC_PROJECT_NUM}@container-engine-robot.iam.gserviceaccount.com" \
  --role=roles/compute.networkUser

gcloud compute networks subnets add-iam-policy-binding "$CP_SUBNET" \
  --project="$HOST_PROJECT" --region="$REGION" \
  --member="serviceAccount:service-${SVC_PROJECT_NUM}@container-engine-robot.iam.gserviceaccount.com" \
  --role=roles/compute.networkUser

gcloud compute networks subnets add-iam-policy-binding "$NODE_SUBNET" \
  --project="$HOST_PROJECT" --region="$REGION" \
  --member="serviceAccount:${SVC_PROJECT_NUM}@cloudservices.gserviceaccount.com" \
  --role=roles/compute.networkUser

gcloud compute networks subnets add-iam-policy-binding "$CP_SUBNET" \
  --project="$HOST_PROJECT" --region="$REGION" \
  --member="serviceAccount:${SVC_PROJECT_NUM}@cloudservices.gserviceaccount.com" \
  --role=roles/compute.networkUser

gcloud compute networks subnets add-iam-policy-binding "$NODE_SUBNET" \
  --project="$HOST_PROJECT" --region="$REGION" \
  --member="user:${CREATE_VM_ACCOUNT}" --role=roles/compute.networkUser

gcloud compute networks subnets add-iam-policy-binding "$CP_SUBNET" \
  --project="$HOST_PROJECT" --region="$REGION" \
  --member="user:${CREATE_VM_ACCOUNT}" --role=roles/compute.networkUser

```

### 3-C. SVC 프로젝트 Node SA 및 권한 — admin@sonmap.net

기존 SA가 있으면 create를 생략한다. SA 생성 및 IAM 변경은 admin@sonmap.net이 수행한다.
cho@sonmap.net에는 SVC Owner 권한이 없으므로 아래 세 가지 사용자 바인딩을 모두 적용한다.
SA 생성이 실패하면 이후 권한 부여를 진행하지 않는다. 기존 터미널에서는 SVC_OWNER를 반드시 다시 정의한다.

```bash
export SVC_OWNER="admin@sonmap.net"
export CREATE_VM_ACCOUNT="cho@sonmap.net"
```

```bash
gcloud auth login "$SVC_OWNER"
gcloud iam service-accounts create sa-l2comm-node --project="$SVC_PROJECT"
gcloud projects add-iam-policy-binding "$SVC_PROJECT" \
  --member="serviceAccount:${NODE_SA}" --role=roles/container.defaultNodeServiceAccount --condition=None
# cho는 SVC Owner가 아니므로 다음 세 역할을 모두 부여한다.
gcloud projects add-iam-policy-binding "$SVC_PROJECT" \
  --member="user:${CREATE_VM_ACCOUNT}" --role=roles/container.admin --condition=None
gcloud projects add-iam-policy-binding "$SVC_PROJECT" \
  --member="user:${CREATE_VM_ACCOUNT}" --role=roles/serviceusage.serviceUsageConsumer --condition=None
gcloud iam service-accounts add-iam-policy-binding "$NODE_SA" \
  --project="$SVC_PROJECT" --member="user:${CREATE_VM_ACCOUNT}" --role=roles/iam.serviceAccountUser --condition=None
```

### 3-D. Host 방화벽 자동 관리 권한 — admin@sonmap.net (선택)

Shared VPC 방화벽 자동 관리를 허용하는 운영 방식을 사용할 경우, SVC의 GKE 서비스 에이전트에 다음 권한을 적용한다.
compute.securityAdmin은 Host 전체 방화벽에 대한 넓은 권한이다. 조직에서 허용하지 않으면 Host 관리자가 공식 GKE 필수 방화벽을 수동 관리하는 방식으로 대체한 후 생성한다.

```bash
gcloud auth login "$HOST_OWNER"
gcloud projects add-iam-policy-binding "$HOST_PROJECT" \
  --member="serviceAccount:service-${SVC_PROJECT_NUM}@container-engine-robot.iam.gserviceaccount.com" \
  --role=roles/compute.securityAdmin
```

## 4. l2comn-dev — GKE만 생성

```bash
gcloud auth login "$CREATE_VM_ACCOUNT"
gcloud container clusters create-auto "$CLUSTER" \
  --project="$SVC_PROJECT" --region="$REGION" \
  --network="projects/${HOST_PROJECT}/global/networks/${NETWORK}" \
  --subnetwork="projects/${HOST_PROJECT}/regions/${REGION}/subnetworks/${NODE_SUBNET}" \
  --cluster-secondary-range-name="$POD_RANGE" \
  --private-endpoint-subnetwork="projects/${HOST_PROJECT}/regions/${REGION}/subnetworks/${CP_SUBNET}" \
  --enable-private-nodes --enable-private-endpoint \
  --enable-master-authorized-networks \
  --no-enable-authorized-networks-on-private-endpoint \
  --service-account="$NODE_SA" --scopes=cloud-platform \
  --release-channel="$RELEASE_CHANNEL"

gcloud container clusters describe "$CLUSTER" --project="$SVC_PROJECT" --region="$REGION"
```

--enable-private-endpoint 사용 시 API가 요구하는 --enable-master-authorized-networks를 함께 지정한다.
--no-enable-authorized-networks-on-private-endpoint는 유지한다. 관리 VM IP는 승인 네트워크에 등록하지 않고 Private Endpoint에 승인 네트워크 제한을 강제하지 않는 설계다.
TCP 443 경로와 IAM/Kubernetes 인증은 별도로 필요하다. 다른 리전에서 접속하면 global access 필요 여부를 확인한다.
위 설정은 사설 Endpoint에 도달 가능한 다른 호스트도 네트워크 차원에서 접근할 수 있게 하며, API 작업에는 인증·권한이 필요하다.

## 5. Edge Owner user01@sonmap.net — AR와 빌드 권한

```bash
gcloud auth login "$EDGE_OWNER"
gcloud artifacts repositories create "$REPO_NAME" --project="$EDGE_PROJECT" \
  --location="$REGION" --repository-format=docker
gcloud iam service-accounts create sa-l2comm-build --project="$EDGE_PROJECT"

gcloud artifacts repositories add-iam-policy-binding "$REPO_NAME" \
  --project="$EDGE_PROJECT" --location="$REGION" \
  --member="serviceAccount:${BUILD_SA}" --role=roles/artifactregistry.writer
gcloud projects add-iam-policy-binding "$EDGE_PROJECT" \
  --member="serviceAccount:${BUILD_SA}" --role=roles/logging.logWriter
gcloud projects add-iam-policy-binding "$EDGE_PROJECT" \
  --member="serviceAccount:${BUILD_SA}" --role=roles/storage.objectViewer

gcloud artifacts repositories add-iam-policy-binding "$REPO_NAME" \
  --project="$EDGE_PROJECT" --location="$REGION" \
  --member="serviceAccount:${NODE_SA}" --role=roles/artifactregistry.reader
gcloud artifacts repositories add-iam-policy-binding "$REPO_NAME" \
  --project="$EDGE_PROJECT" --location="$REGION" \
  --member="user:${EXEC_ACCOUNT}" --role=roles/artifactregistry.reader

# 빌드 제출자는 Edge Owner user01이다. cho에게 빌드 권한을 추가하는 단계는 사용하지 않는다.
```

l2comn-dev에서 저장소 clone 이후 Edge Owner user01@sonmap.net으로 실행한다. VM 위치와 빌드 제출 계정은 별개다.
GitHub Trigger 연결을 새로 만들지 않는 단순 수동 빌드 방식이다.
cho로 제출하여 gcp-sbx-edp-gke01_cloudbuild 버킷 접근이 거부된 경우 user01로 전환한 뒤 다시 실행한다.
user01로도 거부되면 실제 활성 계정과 버킷·조직 정책을 확인한다. --no-source로 우회하지 않는다.
커스텀 Build SA의 storage.objectViewer는 소스 다운로드용이며 필요 시 staging bucket 범위로 축소한다.

```bash
gcloud auth login "$EDGE_OWNER"
gcloud config set account "$EDGE_OWNER"
gcloud auth list --filter=status:ACTIVE
gcloud builds submit . --project="$EDGE_PROJECT" --region="$REGION" \
  --config=cloudbuild/cloudbuild.yaml --substitutions="_IMAGE_URI=${DOCKER_IMAGE}"
gcloud artifacts docker images list "${REGION}-docker.pkg.dev/${EDGE_PROJECT}/${REPO_NAME}" \
  --project="$EDGE_PROJECT" --include-tags
```

### 5-A. Edge AR 이미지 재지정 및 이미지 없음 복구

| 구분 | 지정 값 |
|---|---|
| AR 프로젝트 (Edge) | gcp-sbx-edp-gke01 |
| GKE 프로젝트 | gcp-sbx-edp-comn-509423 |
| 이미지 전체 주소 | asia-northeast3-docker.pkg.dev/gcp-sbx-edp-gke01/ar-l2comm-python/python-bq-batch:v1 |
| 빌드 제출 사용자 | user01@sonmap.net |
| 실제 빌드 실행 SA | cloudbuild.yaml의 sa-l2comm-build |
| Pod 이미지 다운로드 주체 | GKE Node SA (KSA나 .oefj 계정이 아님) |

AR 저장소 생성만으로 이미지는 등록되지 않는다. Listed 0 items는 저장소가 비었다는 뜻이다.
Pod 이벤트의 not found는 요청한 이미지/태그를 찾지 못한 경우다. 403이나 timeout이면 권한·연결을 별도로 확인한다.

```bash
export DOCKER_IMAGE="asia-northeast3-docker.pkg.dev/gcp-sbx-edp-gke01/ar-l2comm-python/python-bq-batch:v1"

gcloud artifacts docker images list \
  asia-northeast3-docker.pkg.dev/gcp-sbx-edp-gke01/ar-l2comm-python \
  --project=gcp-sbx-edp-gke01 --include-tags
```

이미지 또는 v1 태그가 없을 때만 위 5절의 user01 빌드를 실행한다. SUCCESS 후 다시 목록을 확인한다.
등록된 동일 이미지로 재실행할 때는 이미지 빌드를 반복하지 않는다.
실제 Pod 요청 주소도 확인한다. POD_NAME은 조회하려는 실제 이름으로 지정한다.

```bash
kubectl get pods -n l2comm-batch
export POD_NAME="l2comm-20261024-120000-20261024122226"
kubectl get pod "$POD_NAME" -n l2comm-batch \
  -o jsonpath='{.spec.containers[0].image}{"\n"}'
kubectl describe pod "$POD_NAME" -n l2comm-batch
```

DOCKER_IMAGE를 export하면 새 run-gke.sh 실행에 적용된다. 이미 생성된 Pod의 이미지 주소는 바뀌지 않는다.
기존 Pod가 같은 주소로 ImagePullBackOff 상태이면 이미지 등록 후 자동 재시도한다.
.oefj만 수정한 경우는 이미지 재빌드가 필요 없다. 키는 NFS에서 읽는다.

## 6. 두 VM 모두 — Git 소스 배치

l2comn-dev와 central02 각각의 터미널에서 수행한다. VM에 직접 원격 배포하는 단계가 아니라 사용자가 실행할 명령이다.
GitHub 인증은 기존 계정/credential helper를 사용한다.

```bash
git clone https://github.com/sonmap/Gcp_Managed_GKE_L2Comm03-01_re.git
cd Gcp_Managed_GKE_L2Comm03-01_re
git log -1 --oneline
```

이미 clone했으면 해당 디렉터리에서 git pull --ff-only를 사용한다.
l2comn-dev는 소스 보관과 GKE 생성/빌드만 수행하며 mnt 마운트와 NFS 설정은 하지 않는다.

## 7. central02만 — 실행 파일 배치와 NFS

실제 IP와 Zone을 확인한 후 공통 기본 NFS_SERVER와 일치시킨다.

```bash
gcloud compute instances describe central02 --project="$EDGE_PROJECT" --zone="$VM_ZONE" \
  --format='value(networkInterfaces[0].networkIP)'
```

central02 내부에서 실행한다. /ssw/dlk가 기존 다른 경로이면 먼저 확인하며 덮어쓰지 않는다.
install-vm.sh는 사용하지 않는다.

```bash
chmod +x ssw/dlk/run-gke.sh ssw/dlk/client_gke ssw/dlk/setup-nfs-server.sh
chmod +x mnt/batch_gke/l2_gke_sh/rsvp/GP_L2RSVP_001_1/run-gke.sh
sudo mkdir -p /ssw
sudo ln -s "$PWD/ssw/dlk" /ssw/dlk
sudo env NFS_CLIENT_CIDR="$NODE_CIDR" bash ssw/dlk/setup-nfs-server.sh
findmnt /srv/l2comm-nfs
sudo exportfs -v
```

NFS 스크립트는 central02의 저장소 mnt를 /srv/l2comm-nfs에 bind mount한다.
NFSv4 fsid=0을 사용하며 Pod는 /batch_gke/l2/rsvp를 /home/jupyter에 읽기 전용 마운트한다.
.oefj는 기존 승인된 키를 central02의 mnt/batch_gke/l2/rsvp/.oefj에 안전하게 배치한다. Git에 추가하지 않는다.
root_squash 및 Pod 실행 UID 기준으로 읽기 가능한지 확인한다. 키 생성·조직 정책 변경은 수행하지 않는다.

central02 태그는 Edge Owner user01@sonmap.net이, Host 방화벽은 admin@sonmap.net이 실행한다.
IAP SSH 규칙은 기존 규칙을 확인한 후 필요한 경우만 추가한다.

```bash
gcloud auth login "$EDGE_OWNER"
gcloud compute instances add-tags central02 --project="$EDGE_PROJECT" --zone="$VM_ZONE" \
  --tags=iap-ssh-central02,l2comm-nfs

# Host Owner
gcloud auth login "$HOST_OWNER"
gcloud compute firewall-rules create allow-l2comm-nfs-from-gke --project="$HOST_PROJECT" \
  --network="$NETWORK" --direction=INGRESS --priority=1000 --action=ALLOW \
  --rules=tcp:2049 --source-ranges="$NODE_CIDR" --target-tags=l2comm-nfs
gcloud compute firewall-rules create allow-iap-ssh-central02 --project="$HOST_PROJECT" \
  --network="$NETWORK" --direction=INGRESS --priority=1000 --action=ALLOW \
  --rules=tcp:22 --source-ranges=35.235.240.0/20 --target-tags=iap-ssh-central02
```

NFS 마운트는 Node에서 발생하므로 Node CIDR을 허용한다. 상위 deny 및 OS 방화벽도 확인한다.
central02는 Spot VM이므로 정지·회수 시 NFS도 중단된다. 재시작 후 bind mount/export 상태를 확인한다.
이 구성에서 Filestore는 새로 생성하지 않는다.

### 7-A. .oefj 배치·확인 — central02만

키 내용은 Git, 로그, 채팅에 출력하지 않는다. 기존 승인된 키를 사용하며 새 키 생성은 하지 않는다.

| 위치 | 경로 |
|---|---|
| central02 원본 | ~/Gcp_Managed_GKE_L2Comm03-01_re/mnt/batch_gke/l2/rsvp/.oefj |
| central02 bind mount | /srv/l2comm-nfs/batch_gke/l2/rsvp/.oefj |
| NFS에서 공유하는 하위 경로 | /batch_gke/l2/rsvp |
| Pod에서 읽는 경로 | /home/jupyter/.oefj |

l2comn-dev에만 복사하면 central02의 NFS에 반영되지 않는다.
central02에서 기존 파일을 덮어쓸지 확인한 뒤, 승인된 전달 방법으로 키를 위 원본 경로에 복사한다.
아래 SOURCE_KEY는 central02에 이미 안전하게 전달된 파일의 실제 경로로 변경한다.

```bash
cd ~/Gcp_Managed_GKE_L2Comm03-01_re
export SOURCE_KEY="/approved/path/to/.oefj"
cp -i -- "$SOURCE_KEY" mnt/batch_gke/l2/rsvp/.oefj
```

이미 복사했다면 cp를 생략하고 파일 존재·소유자·권한·bind mount만 확인한다.

```bash
ls -l mnt/batch_gke/l2/rsvp/.oefj
findmnt /srv/l2comm-nfs
ls -l /srv/l2comm-nfs/batch_gke/l2/rsvp/.oefj
namei -l /srv/l2comm-nfs/batch_gke/l2/rsvp/.oefj
git check-ignore mnt/batch_gke/l2/rsvp/.oefj
```

git check-ignore 출력이 없으면 키를 add/commit하지 말고 제외 규칙을 먼저 확인한다.
Missing or unreadable SA key는 파일 부재 또는 읽기 권한 문제다.
NFS root_squash 환경에서는 Pod의 root도 파일 소유자의 root 권한을 그대로 사용하지 못한다.
Pod의 실제 UID/GID, 키 소유자와 읽기 권한, 상위 디렉터리 탐색 권한을 함께 확인한다.
무조건 chmod 777로 해결하지 않는다. 승인된 UID/GID·그룹·ACL 정책에 맞춰 읽기 권한을 부여한다.
Failed Pod는 파일 복사 후 자동 재실행되지 않으므로 9-A 절차로 Pod만 삭제하고 재실행한다.

## 8. SVC 관리자 admin@sonmap.net — central02 실행 사용자의 GKE 권한

기존 client_gke는 Namespace/KSA/PV/PVC/Pod를 생성·적용한다. PV와 Namespace는 클러스터 범위 리소스다.
변경을 최소화하는 PoC로 container.admin을 부여한다. 운영에서는 초기 생성과 실행을 분리하여 권한을 축소한다.

```bash
gcloud auth login "$SVC_OWNER"
gcloud projects add-iam-policy-binding "$SVC_PROJECT" \
  --member="user:${EXEC_ACCOUNT}" --role=roles/container.admin
```

central02에서 실행한다. VM SSH 인증 권한과 GKE 권한은 별개다.

```bash
gcloud auth login "$EXEC_ACCOUNT"
export KUBECONFIG="$HOME/.kube/config_new-autopilot"
gcloud container clusters get-credentials "$CLUSTER" \
  --project="$SVC_PROJECT" --region="$REGION" --internal-ip
kubectl auth can-i create persistentvolumes
kubectl auth can-i create pods --namespace=l2comm-batch
kubectl get nodes -o wide
```

## 9. central02 — 기존 run-gke.sh 실행

```bash
export NFS_SERVER="172.32.10.4"
export NFS_PATH="/batch_gke/l2/rsvp"
export BQ_JOB_PROJECT="pjt-c-admin"
export SA_KEY_PATH="/home/jupyter/.oefj"
export DRY_RUN=1

./mnt/batch_gke/l2_gke_sh/rsvp/GP_L2RSVP_001_1/run-gke.sh \
  20261024 20261024 20261024 120000 \
  0 4 16 \
  /mnt/batch_gke/l2/rsvp/GP_L2RSVP_001_1/python \
  python bq_auth_smoke.py
```

출력된 image/NFS/CPU/memory/인증 경로를 확인한 후 DRY_RUN=0으로 같은 명령을 실행한다.
프로그램은 Pod를 직접 생성하고 성공 로그를 가져온 후 Pod를 삭제한다. Job으로 변경하지 않는다.
기존 PV/PVC의 server/path 변경은 불변 필드 제약을 확인한다. 자동 삭제·재생성하지 않는다.
키 소유 SA에는 기존 BigQuery job/data 권한이 필요하다. BQ 인증 방식은 유지한다.

### 9-A. .oefj 보완 후 실패한 Pod 삭제·재실행

이미지 다운로드가 성공한 뒤 Exit Code 2와 Missing or unreadable SA key가 나오면 7-A부터 확인한다.
PV/PVC와 이미지는 유지하고 실패한 Pod만 삭제한다. kubectl 권한이 있는 VM에서 실행한다.
삭제 대상 이름과 Failed 상태를 먼저 확인한다.

```bash
export FAILED_POD="l2comm-20261024-120000-20261024122226"
kubectl get pod "$FAILED_POD" -n l2comm-batch
kubectl delete pod "$FAILED_POD" -n l2comm-batch
```

central02에서 user01로 GKE 접속을 준비한 뒤 기존 wrapper를 실행한다.
0절 변수가 없는 새 터미널에서도 아래의 명시적 값으로 접속할 수 있다.

```bash
cd ~/Gcp_Managed_GKE_L2Comm03-01_re
gcloud auth login user01@sonmap.net
export KUBECONFIG="$HOME/.kube/config_new-autopilot"
gcloud container clusters get-credentials gkea-prod-l2comn-batch \
  --project=gcp-sbx-edp-comn-509423 --region=asia-northeast3 --internal-ip

export DOCKER_IMAGE="asia-northeast3-docker.pkg.dev/gcp-sbx-edp-gke01/ar-l2comm-python/python-bq-batch:v1"
export NFS_SERVER="172.32.10.4"
export NFS_PATH="/batch_gke/l2/rsvp"
export SA_KEY_PATH="/home/jupyter/.oefj"
export BQ_JOB_PROJECT="pjt-c-admin"
export DRY_RUN=0
export RUN_YMD="$(date +%Y%m%d)"
export RUN_TIME="$(date +%H%M%S)"

./mnt/batch_gke/l2_gke_sh/rsvp/GP_L2RSVP_001_1/run-gke.sh \
  "$RUN_YMD" "$RUN_YMD" "$RUN_YMD" "$RUN_TIME" \
  0 4 16 \
  /mnt/batch_gke/l2/rsvp/GP_L2RSVP_001_1/python \
  python bq_auth_smoke.py
```

NFS_SERVER는 central02의 실제 내부 IP와 일치시킨다.
成功時は既存client_gkeがログを表示してPodを削除する。失敗時は新しいPod名でdescribe/logsを確認する。
この手順はPod削除・再実行用の文書であり、Git更新時に実機のPod削除やキーコピーを自動実行しない。

## 10. 확인

```bash
gcloud container clusters describe "$CLUSTER" --project="$SVC_PROJECT" --region="$REGION" \
  --format='yaml(name,network,subnetwork,ipAllocationPolicy,privateClusterConfig,nodeConfig.serviceAccount)'
kubectl get pv
kubectl get pvc,pods -n l2comm-batch -o wide
```

실환경 미검증 항목: IP 중복·여유, 조직 정책, 할당량, API 연결, 실제 VM IP, IAM 반영, 패키지 다운로드 경로, NFS/BQ 실행.
원본 보존: archive/original에 원본 파일을 보관한다. 기존 gcloud/*.sh는 실행 대상이 아니다.

공식 자료:
- https://docs.cloud.google.com/kubernetes-engine/docs/how-to/cluster-shared-vpc
- https://docs.cloud.google.com/sdk/gcloud/reference/container/clusters/create-auto
- https://docs.cloud.google.com/artifact-registry/docs/integrate-gke
