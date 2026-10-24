# 검증 결과

- 원본 tree: 5d620b0061c7e0dc013a2fa38b21a2052e058736
- 원본 35개 파일을 archive/original에 내용 그대로 보존.
- 활성 실행 파일 6개 bash -n 통과.
- 업무별 wrapper DRY_RUN=1 실행 통과.
- 생성 manifest의 AR 프로젝트 gcp-sbx-edp-gke01, NFS 172.32.10.4, /home/jupyter/.oefj 및 업무 디렉터리 변환 확인.
- 클라우드 생성, VM 배포, 실제 NFS mount 및 BigQuery 실행은 수행하지 않음.
