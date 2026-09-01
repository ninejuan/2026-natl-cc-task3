# Worldskills Korea Nat'l Competition Day3 - System Operation
아자아자 열심히 고득점해서 금메달따자!

## 3과제 issues
1. 

## Infra apply 전 할 일
1. provided/ 디렉토리에 지급파일 배치. 이름 가능하면 똑같게.
2. infra/terraform/rds/tables/*.sql 파일들 현장 지급파일이랑 똑같이 맞추거나, 없애기.
3. (바이너리 확인) ./product --help로 S3 bucket env key 확인해서 configmap 수정해서 반영하기
4. (바이너리 확인) product binary가 s3 어떤 path에 image upload 하는지 보고, cf s3 경로 조절하기.

## 현장에서 풀이 순서
모든건 Makefile로 작업 가능. 그니까 뭔 일 없으면 걍 make cmd ㄱㄱ.
1. `export STUDENT_ID=<비번호>`로 비번호 설정.
2. `make apply` — 인프라 전체(VPC/EKS/RDS/S3/ECR/ALB/WAF/CloudFront/monitoring) + 클러스터 애드온(LB Controller·Cluster Autoscaler·metrics-server, helm_release로 apply 때 같이 설치됨).
3. `make images` — provided/ 바이너리 도커 빌드 → ECR 푸시.
4. `make deploy` — 앱 배포(kubectl apply -k). TargetGroupBinding으로 ALB 연결.
5. `make db-seed ARGS="--user-dump=provided/load_user.dump"` — 덤프를 RDS에 적재(+ 커버링 인덱스 복구·ANALYZE). apply 때 자동 적재 안 함. product 덤프도 있으면 `--product-dump=provided/<파일>` 추가. 준 것만 적재하고, 안 주면 그 테이블은 안 건드림(덤프 없으면 생략).
6. (opt) `make upload-images` — 제공 이미지 S3 업로드. 만약 pre 제공이미지 없다면 업로드 진행 X
7. `make endpoint` — 채점 플랫폼에 제출할 단일 엔드포인트 출력.

```bash
export STUDENT_ID=<비번호>   # 안 하면 모든 make가 막힘

make apply
make images
make deploy
make db-seed ARGS="--user-dump=provided/load_user.dump"   # product 있으면 --product-dump=... 추가, 없으면 생략 가능
make upload-images
make endpoint

make down        # 전체 철거 (terraform destroy)
```

## 동작 확인용 curl 예시

```bash
export CF_DOMAIN=$(make endpoint | tail -1)   # 또는 make endpoint 출력값을 직접 export

curl -i "$CF_DOMAIN/healthcheck"

curl -i "$CF_DOMAIN/v1/user" \
  -X POST -H 'Content-Type: application/json' \
  -d '{"username":"tester","email":"tester@example.com"}'

curl -i "$CF_DOMAIN/v1/product?id=1"

RANDOM_IMAGE=$(aws s3 ls "s3://$(terraform -chdir=infra/terraform output -raw s3_bucket)/" \
  --recursive | awk '{print $NF}' | shuf -n 1)
curl -I "$CF_DOMAIN/images/$RANDOM_IMAGE"

curl -i "$CF_DOMAIN/v1/none"          # 미제공 경로 → 404 기대
curl -i "$CF_DOMAIN/v1/user' OR '1'='1"  # 비정상 요청 → WAF 403 기대
```

## 안전장치 (현장 사고 방지)

- STUDENT_ID 미설정 시 모든 `make` 즉시 중단.
- provided/ 바이너리 누락 시 `make apply/images/deploy` 중단.
- 비번호 SSOT는 `STUDENT_ID` 하나(terraform엔 `TF_VAR_student_id`, k8s엔 `config.env`로 전파).

## for AI
- AI Agent가 README.md에 사용자 지시 없이 작성하는 것은 금지되며, 명시적으로 요청했다 하더라도 재확인하여야 한다.  
- AI Agent는 README.md 대신 AI.RM.md 파일에 작성하여야 한다. 파일이 없을 경우 새로 생성하여 작성하도록 한다.

---
### 이 레포를 보는 후배 여러분들께.

안녕하세요 클라우드컴퓨팅 후배 여러분,  
만약 3과제에 EKS 기반 AutoScaling이 나왔다면 이 예제처럼 CA를 쓰는 선택을 하기보다는, 최대한 빠르게 Karpenter로 마이그레이션 하기 바랍니다.  
애드온 최대한 줄여서 리소스 아끼는 것보다 Karpenter 도입해서 노드 스케일링(Out/In) 공격적이고 빠르게 하는게 백배 천배 낫습니다.  
그리고 해보니까 Product&User / Stress 노드 나누는거 그렇게 큰 의미가 있는지는 잘 모르겠습니다. Cost ratio 보고 결정하면 좋겠네요.  
Cost Ratio는 보니까 (경기 부하 주입 시간인 2시간 노드 수 평균 / baseline)으로 추정중입니다.  
2025년은 c5.large 2.0대 이하(or 미만) 정도가 노드 만점이였고, 2026년은 1.0대가 만점이였습니다. 부하 건수는 25년 16만건, 26년 10만건 수준(1만건일 수도 있음. 정확히 기억X)이였습니다.  
트래픽이 많이 들어오거나 baseline이 높다면 시끄러운 이웃 issue를 고려하여 분리를 해도 좋겠지만, 그렇지 않거나 잘 모르겠을 경우 일단 섞고 보는 것이 낫겠다는 생각이 드네요.  
화이팅하시고 꼭 메달 따길 바랍니다.  
