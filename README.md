# Gcp_Managed_GKE_L2Comm03-01_re

원본 sonmap/Gcp_Managed_GKE_L2Comm03-01에서 재구성한 4개 프로젝트 PoC.

| 프로젝트 | 역할 |
|---|---|
| gcp-prod-edp-hub-vpchost | Shared VPC / 서브넷 / 방화벽 |
| gcp-sbx-edp-gke01 | central02 / AR / NFS / 배치 실행 |
| gcp-sbx-edp-comn-509423 | GKE Autopilot |
| gcp-prod-edp-edge-509423 | l2comn-dev / GKE 생성 작업 |

**[실행 절차: docs/RUNBOOK.md](docs/RUNBOOK.md)** 를 순서대로 사용한다.

- 인프라는 문서의 개별 gcloud 명령으로 구성한다.
- 두 VM 모두 이 저장소를 clone한다.
- NFS 설정 shell은 central02에서만 실행하며 저장소 mnt를 bind mount한다.
- 기존 run-gke.sh → client_gke → Pod 생성 및 .oefj 인증을 재사용한다.
- 기존 인프라 shell과 원본 파일은 archive/original에 참고용으로 보존한다. 실행하지 않는다.
- VM/클라우드 리소스에 자동 적용하지 않는다. 문서의 명령은 사용자가 확인 후 실행한다.
- CIDR은 중복 검증 전 후보이며 central02 IP는 실값 확인이 필요하다.
- central02는 Spot VM이므로 중단되면 NFS도 중단된다.

출처: 원본 tree 5d620b0061c7e0dc013a2fa38b21a2052e058736. 업무 Python·Dockerfile 및 업무별 wrapper는 원본을 재사용한다.
