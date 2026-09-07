# IWTC NestJS 백엔드 신규 구축 계획

## 1. 문서 목적

이 문서는 기존 Spring Boot 백엔드의 기능과 API 동작을 참고해, NestJS 기반의 새로운 IWTC 백엔드를 구축하는 계획을 기록한다.

새 백엔드는 기존 운영 데이터베이스와 데이터를 이전하지 않는다. 데이터베이스, 계정, 비밀값, 배포 리소스를 모두 새로 만들고 서비스 데이터도 처음부터 쌓는다.

기존 Spring 백엔드는 다음 용도로만 사용한다.

- 프론트엔드가 사용 중인 API 경로와 응답 형식 확인
- 게임, 랭킹, 인증 등 기존 비즈니스 규칙 확인
- 새로운 NestJS 구현의 계약 테스트 작성

기존 Spring 서버와 데이터베이스는 새 서비스의 운영 구성이나 롤백 수단으로 사용하지 않는다.

## 2. 확정된 전환 원칙

1. 기존 운영 DB에 연결하거나 데이터를 가져오지 않는다.
2. 기존 저장소에 남아 있는 DB, AWS, JWT 등의 비밀값을 재사용하지 않는다.
3. 새 PostgreSQL 데이터베이스와 새 서비스 계정을 생성한다.
4. 기존 DB 스키마를 복제하지 않고 서비스 요구사항을 기준으로 새로 설계한다.
5. 프론트엔드가 의존하는 API 계약과 핵심 비즈니스 규칙은 최대한 유지한다.
6. 회원 API와 일반 API를 하나의 NestJS 애플리케이션으로 통합한다.
7. 배포는 현재 홈서버의 GitHub Actions, GHCR, Argo CD, Kubernetes, Traefik 구성을 따른다.
8. 운영 롤백은 Spring 서버가 아니라 이전 NestJS 이미지로 수행한다.
9. 비밀값은 Git, Dockerfile, Docker 이미지, Docker 빌드 인자에 넣지 않는다.

## 3. 기술 방향

### 애플리케이션

- 런타임: Node.js 24.15 이상인 24.x LTS
- 프레임워크: NestJS 12.x
- 언어: 호환되는 최신 TypeScript
- ORM: Prisma ORM 8.x
- 데이터베이스: PostgreSQL 18.x
- API: REST, OpenAPI 문서화
- 구조: 단일 Docker 이미지로 배포하는 모듈형 모놀리스
- 패키지 설치: lock file을 사용하는 재현 가능한 설치

NestJS 12는 ESM 관련 변경과 CLI의 Node.js 최소 버전 조건이 있으므로 프로젝트 생성 시 모듈 형식을 명시하고, 로컬·CI·Docker에서 같은 Node.js 버전을 사용한다.

PostgreSQL Docker 이미지는 `postgres:18.6`처럼 검증한 minor 버전까지 고정하고 `postgres:latest`를 사용하지 않는다. 같은 major 안에서는 보안·버그 수정이 포함된 최신 minor로 정기 갱신한다.

### 데이터

- 데이터베이스: 홈서버 Host Docker에서 실행하는 새 PostgreSQL
- ORM: Prisma ORM 7.10 안정판
- 스키마 관리: Prisma Schema와 SQL migration
- 초기 데이터: 버전 관리되는 seed
- 캐시: 필요성이 확인되기 전까지 Redis를 사용하지 않음
- 미디어: S3 또는 S3 호환 오브젝트 스토리지

기존 DB를 사용하지 않으므로 introspection은 수행하지 않는다. Prisma Schema를 새로 설계하고 첫 migration부터 변경 이력을 관리한다.

프로젝트 생성 시점에 npm의 Prisma 8은 RC 버전만 제공되므로 운영 기준은 Prisma 7.10 안정판으로 고정한다. Prisma 8 정식판이 나온 뒤 업그레이드 호환성을 별도로 검증한다.

Prisma 7의 기본 변경 흐름은 다음과 같다.

```text
schema.prisma 수정
  -> prisma migrate dev
  -> migration 검토 및 버전 관리
  -> prisma migrate deploy
  -> prisma migrate status
```

## 4. 목표 시스템 구성

```text
Internet
   |
Traefik Ingress (HTTPS/TLS)
   |
NestJS Service (ClusterIP)
   |
NestJS Deployment (2 replicas)
   |-- PostgreSQL (Host Docker, private network)
   `-- S3-compatible object storage
```

- 외부 요청은 Traefik의 80/443 포트로만 받는다.
- NestJS와 PostgreSQL 포트를 인터넷에 직접 공개하지 않는다.
- NestJS는 `https://api.<domain>` 형태의 별도 API 호스트로 제공한다.
- TLS 인증서는 현재 운영 중인 cert-manager와 Let's Encrypt ClusterIssuer를 사용한다.
- 애플리케이션 이미지는 홈서버에서 빌드하지 않고 GitHub Actions에서 빌드한다.
- GHCR에는 `latest`와 commit SHA 태그를 올리되 실제 배포에는 변경 불가능한 commit SHA 태그를 사용한다.

### PostgreSQL 운영 방향

초기에는 상태 비저장 NestJS 애플리케이션만 Argo CD로 배포하고, PostgreSQL 18은 홈서버의 별도 Docker 컨테이너로 운영한다.

```text
Ubuntu Home Server
├── K3s
│   ├── Traefik
│   ├── NestJS Pod #1
│   ├── NestJS Pod #2
│   └── Service
└── Docker
    └── PostgreSQL 18
        └── /srv/iwtc/postgresql
```

PostgreSQL 데이터는 Kubernetes PersistentVolume이 아니라 `/srv/iwtc/postgresql` host bind mount에 저장한다. 해당 경로의 소유권, 권한, 디스크 여유 공간을 배포 전에 확인한다.

PostgreSQL의 5432 포트는 공유기에서 포트 포워딩하지 않는다. Host에서도 `0.0.0.0:5432`로 무조건 공개하지 않고 K3s가 접근할 수 있는 내부 interface와 방화벽 규칙으로 제한한다.

방화벽에 허용할 CIDR은 문서에 미리 하드코딩하지 않는다. 실제 구축 시 K3s/flannel 구성과 Pod에서 Host로 접속할 때 보이는 source address를 확인한 뒤 허용 범위를 정한다.

### K3s에서 Host PostgreSQL 연결

NestJS 설정에 Host IP를 직접 넣지 않고 selector 없는 Kubernetes Service와 EndpointSlice로 연결을 추상화한다.

```text
NestJS Pod
  -> postgresql:5432
  -> Kubernetes Service (selector 없음)
  -> EndpointSlice
  -> Ubuntu Host Internal IP:5432
  -> Docker PostgreSQL
```

애플리케이션의 `DATABASE_URL`에는 `postgresql` Service 이름을 사용한다. 실제 Host IP는 EndpointSlice에서만 관리한다. EndpointSlice에는 loopback 주소가 아닌 홈서버의 고정 내부 IP를 지정한다.

### 단일 노드와 replicas의 한계

2 replicas는 무중단 RollingUpdate와 Pod·프로세스 단위 장애 대응을 위한 것이다. 단일 홈서버 노드, SSD, 전원 장애에 대한 HA는 제공하지 않는다.

단일 홈서버 장애에 대비하기 위해 DB 백업은 반드시 홈서버 외부 위치에도 보관한다.

## 5. 저장소 운영 방향

점진적인 구현과 코드 비교를 위해 저장소는 다음처럼 분리한다.

```text
iwtc-frontend-new/    # 현재 프론트엔드
iwtc-backend-new/     # 기존 Spring 기능·API 참고용
iwtc-backend-nest/    # 신규 NestJS 백엔드
```

세 저장소는 하나의 Cursor workspace에서 함께 열어 프론트엔드 호출 코드, 기존 Spring 구현, 신규 NestJS 구현을 비교한다.

기존 Spring 저장소에는 신규 NestJS 코드를 섞지 않는다. 이 문서는 NestJS 저장소가 생성되면 해당 저장소로 복사하고 이후 구현 현황도 그곳에서 관리한다.

## 6. 보안 및 소유권 원칙

기존 저장소에 포함된 암호화 값이나 자격 증명은 복구해서 새 서비스에 사용하지 않는다.

새로 생성할 항목은 다음과 같다.

- PostgreSQL 데이터베이스와 NestJS 전용 사용자
- JWT access token 및 refresh token 서명 키
- S3 또는 S3 호환 저장소의 전용 접근 키
- 초기 관리자 계정
- GitHub Actions와 GHCR에 필요한 권한
- Kubernetes Secret

운영 비밀값은 Kubernetes Secret을 통해 런타임에 주입한다. Kubernetes Secret의 base64 값은 암호화가 아니므로 실제 Secret YAML은 Git에 커밋하지 않는다. 저장소에는 키 이름과 사용 방법만 기록한 `secret.example.yaml` 또는 `.env.example`만 보관한다.

초기 홈랩 운영에서는 실제 Secret을 클러스터에 직접 생성한다. 이후 GitOps 기반 Secret 관리가 필요해지면 SOPS, Sealed Secrets 또는 External Secrets 도입을 검토한다.

초기 관리자 계정의 비밀번호도 seed나 Git에 기록하지 않는다. 일회성 초기화 명령이나 배포 시 주입되는 임시 Secret을 통해 생성한 뒤 교체한다.

## 7. 단계별 구축 계획

### 0단계: 신규 서비스 경계 확정

- 기존 DB와 기존 자격 증명을 사용하지 않는다는 원칙 확정
- 신규 저장소 `iwtc-backend-nest` 생성
- API 운영 도메인 결정
- PostgreSQL Host Docker 구성과 bind mount 경로 확정
- S3 또는 S3 호환 저장소 결정
- 운영 환경 변수 목록 작성
- 기존 Spring 저장소는 읽기 전용 참고 대상으로 유지

### 1단계: 기존 API 계약 추출

프론트엔드 코드와 기존 Spring 구현을 바탕으로 다음 항목을 기록한다.

- API 경로와 HTTP 메서드
- path, query, body 파라미터
- 응답 JSON과 상태 코드
- 공통 에러 응답 형식
- 인증 필요 여부
- access token 및 refresh token 전달 방식
- 목록 정렬, 필터, 검색, 페이징 규칙
- 게임 결과와 랭킹 계산 규칙
- 이미지와 미디어 처리 규칙

기존 DB의 실제 응답을 기준으로 삼지 않는다. 재현 가능한 fixture와 seed 데이터를 만들어 계약 테스트에 사용한다.

### 2단계: NestJS 기반 구성

- NestJS 프로젝트 생성
- 환경 변수 스키마와 시작 시 검증
- 전역 입력 검증
- 예외 응답 형식 통일
- 구조화 로그와 요청 ID
- OpenAPI 설정
- `/health/live`, `/health/ready` 구성
- graceful shutdown 구성
- Prisma 7.10과 PostgreSQL 연결
- 첫 Prisma Schema와 migration 작성
- 개발 및 테스트용 seed 작성
- Dockerfile과 로컬 Compose 작성

초기 모듈 후보는 다음과 같다.

- `world-cup`
- `candidate`
- `game`
- `rank`
- `reply`
- `member`
- `auth`
- `manage`
- `media`

### 3단계: 신규 DB 설계

초기 테이블 후보는 다음과 같다.

```text
members
roles / permissions (설계 후 확정)
refresh_tokens
world_cups
candidates
games
game_results
rankings
replies
media_files
```

설계 시 다음 사항을 명시한다.

- 기본 키와 외래 키
- unique 제약 조건
- 조회와 랭킹 계산에 필요한 index
- 생성·수정 시간
- hard delete와 soft delete 기준
- 회원 비밀번호 해시 방식
- member 역할과 권한 모델
- refresh token의 jti, family, hash, rotation, reuse detection 및 폐기 방식
- 미디어 object key와 메타데이터
- 트랜잭션 경계
- 시간대 기준

복잡한 랭킹·집계 쿼리는 Prisma API만 고집하지 않고 TypedSQL 또는 검증된 파라미터 바인딩 쿼리를 사용할 수 있다.

### 4단계: 기능별 구현

다음 순서로 구현한다.

1. 월드컵 공개 목록과 상세 조회
2. 후보 및 게임 데이터 조회
3. 게임 진행, 결과 저장, 클리어 처리
4. 랭킹과 통계
5. 댓글
6. 회원가입, 로그인, 토큰 재발급
7. 미디어 업로드와 조회
8. 관리자 기능

각 기능은 다음 절차를 반복한다.

1. 프론트엔드의 실제 호출 코드를 확인한다.
2. 기존 Spring 코드에서 비즈니스 규칙을 확인한다.
3. 요청·응답 계약 테스트를 작성한다.
4. 새 DB schema와 seed를 작성한다.
5. NestJS 기능을 구현한다.
6. 프론트엔드 개발 환경에서 회귀 테스트한다.

### 5단계: CI/CD와 Kubernetes 구성

신규 저장소에 다음 파일을 구성한다.

```text
.github/workflows/deploy.yml
Dockerfile
k8s/
├── deployment.yaml
├── service.yaml
├── ingress.yaml
├── postgresql-service.yaml
├── postgresql-endpointslice.yaml
├── migration-job.yaml
└── secret.example.yaml
argocd/
└── application.yaml
```

GitHub Actions는 다음 순서로 동작한다.

```text
main 브랜치 push
  -> lint, type check, unit/integration test
  -> 빈 PostgreSQL에서 전체 migration, seed, integration test
  -> Docker 이미지 빌드
  -> GHCR에 latest 및 commit SHA 태그 push
  -> k8s/deployment.yaml의 이미지 태그를 commit SHA로 갱신
  -> 변경된 매니페스트를 main에 commit
  -> Argo CD가 변경 감지 및 자동 동기화
  -> PreSync Prisma Migration Job
  -> Kubernetes Deployment 롤링 업데이트
```

`k8s/**`만 변경된 배포 commit은 중복 이미지 빌드 대상에서 제외한다.

Deployment에는 다음 운영 설정을 포함한다.

- 2 replicas
- readiness probe
- liveness probe
- CPU와 메모리 requests/limits
- rolling update
- graceful shutdown
- `terminationGracePeriodSeconds`
- commit SHA 이미지
- Kubernetes Secret을 통한 환경 변수 주입
- 비공개 GHCR 사용 시 `imagePullSecrets`

Prisma migration은 애플리케이션 Pod가 시작할 때마다 실행하지 않는다. Argo CD PreSync Hook인 별도 Kubernetes Job에서 `prisma migrate status` 확인 후 `prisma migrate deploy`를 실행한다.

```yaml
metadata:
  generateName: iwtc-db-migration-
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

`generateName`으로 매 Sync마다 새 Job을 만들고 성공한 Job은 삭제한다. 실패한 Job은 로그 확인을 위해 남긴다. selective sync에서는 Hook이 실행되지 않으므로 운영 애플리케이션 배포에 selective sync를 사용하지 않는다.

PreSync Job은 다음 순서로 실행한다.

```text
prisma validate
  -> prisma migrate status
  -> prisma migrate deploy
  -> readiness 확인
```

Migration이 실패하면 Sync를 실패 처리하고 신규 Deployment를 진행하지 않는다.

### 6단계: 프론트엔드 연결과 서비스 공개

새 서비스는 다음 순서로 공개한다.

```text
NestJS와 신규 PostgreSQL 준비
  -> migration 실행
  -> 초기 관리자 계정 생성
  -> 기본 서비스 데이터 등록
  -> 프론트엔드 개발 환경 연결
  -> 전체 기능과 모바일 환경 테스트
  -> Traefik Ingress와 TLS 연결
  -> 프론트엔드 운영 API 주소 변경
  -> 서비스 공개
```

현재 분리된 일반 API와 회원 API는 하나의 주소로 합친다.

```dotenv
NEXT_PUBLIC_API_BASE_URL=https://api.<domain>/
```

NestJS 통합이 완료되면 `NEXT_PUBLIC_API_MEMBER_URL`과 프론트엔드의 회원 API 주소 분기 로직을 제거한다.

## 8. DB migration 운영 원칙

새 DB도 운영을 시작한 뒤에는 스키마 변경과 롤백 전략이 필요하다.

- 모든 스키마 변경은 Prisma Schema와 SQL migration으로 버전 관리한다.
- 개발 환경에서는 Schema 변경 후 `prisma migrate dev`로 migration을 생성하고 SQL을 검토한다.
- 운영 환경에서는 검토·커밋된 migration만 PreSync Job의 `prisma migrate deploy`로 적용한다.
- 운영 DB를 직접 수정한 뒤 Schema를 맞추는 방식은 금지한다.
- 운영 DB에 한 번 적용된 migration은 수정하거나 삭제하지 않는다.
- 변경이 필요하면 새로운 forward migration을 추가한다.
- 데이터 삭제나 컬럼 삭제는 즉시 수행하지 않는다.
- 스키마는 이전 애플리케이션 버전과 일정 기간 호환되도록 변경한다.

파괴적인 변경은 다음 순서를 따른다.

```text
새 컬럼·테이블 추가
  -> 새 코드 배포
  -> 데이터 전환 및 검증
  -> 안정화 기간 운영
  -> 사용하지 않는 컬럼·테이블 제거
```

## 9. 롤백 전략

신규 서비스의 운영 롤백 대상은 Spring이 아니라 직전 NestJS 버전이다.

```text
신규 NestJS 버전에서 문제 발생
  -> Deployment 이미지 태그를 이전 commit SHA로 변경
  -> Argo CD 동기화
  -> 이전 NestJS 버전으로 복구
```

애플리케이션 롤백과 데이터베이스 롤백은 서로 다르다. Migration이 성공한 뒤 신규 애플리케이션 배포가 실패해도 DB schema는 자동으로 이전 상태로 돌아가지 않는다.

롤백 시 DB schema가 이전 이미지와 호환되어야 한다. 따라서 배포와 동시에 되돌릴 수 없는 컬럼 삭제나 데이터 변환을 수행하지 않는다.

긴급 장애에 대비해 다음 정보를 문서화한다.

- 직전 정상 이미지 SHA
- 이미지 롤백 방법
- Prisma migration 상태 확인 방법
- DB 백업 복구 방법
- Traefik Ingress 차단 또는 점검 페이지 전환 방법

## 10. PostgreSQL 연결 및 백업 운영

### Connection pool

2개의 NestJS Pod는 각각 별도의 DB connection pool을 가진다. 최초 운영 전에 다음 조건이 성립하도록 pool 크기를 작게 설정하고 부하를 보며 조정한다.

```text
Pod 수 x Pod별 pool
  + Migration Job
  + Backup
  + Admin 작업
  + 운영 여유분
  < PostgreSQL max_connections
```

NestJS 프로세스에서는 Prisma DB client를 전역 단일 인스턴스로 관리하고 요청마다 새 client를 생성하지 않는다.

### 백업

백업 정책에는 다음 항목을 포함한다.

- pg_dump 실행 주기
- 일간·주간 백업 보존 기간
- 홈서버 외부 저장 위치
- 백업 파일 암호화와 암호화 키 관리
- 백업 실패 감지와 알림
- 정기 restore test

백업 검증은 파일 존재 확인으로 끝내지 않는다.

```text
pg_dump
  -> 신규 PostgreSQL에 pg_restore
  -> NestJS 연결
  -> migration 상태와 주요 데이터 검증
```

## 11. 테스트 기준

### 자동 테스트

- 도메인 단위 테스트
- API 계약 테스트
- Prisma/PostgreSQL repository 통합 테스트
- 인증과 권한 테스트
- 랭킹 계산 테스트
- 파일 업로드 테스트
- migration과 seed 테스트
- Docker 이미지 구동 테스트

### 배포 전 확인

- 신규 빈 PostgreSQL에 migration이 처음부터 정상 적용됨
- seed가 여러 번 실행되어도 데이터가 중복되지 않음
- readiness와 liveness probe가 정상 동작함
- Secret이 로그와 이미지에 포함되지 않음
- 프론트엔드 주요 사용자 흐름이 정상 동작함
- 모바일 환경과 HTTPS에서 혼합 콘텐츠 문제가 없음
- DB 백업을 다른 환경에 실제로 복구할 수 있음

## 12. 1차 완료 기준

- 프론트엔드가 사용하는 API 계약 목록이 완성되어 있다.
- 새로운 Prisma Schema와 migration 이력이 존재한다.
- 신규 빈 DB에서 migration과 seed만으로 개발 환경을 재현할 수 있다.
- 주요 공개 조회와 게임 API가 정상 동작한다.
- 회원가입, 로그인, 토큰 재발급이 정상 동작한다.
- 테스트와 Docker 이미지 빌드가 자동화되어 있다.
- GHCR, Argo CD, Kubernetes, Traefik 배포가 자동화되어 있다.
- cert-manager와 Let's Encrypt로 API TLS 인증서가 자동 관리된다.
- 모든 운영 요청이 HTTPS API 도메인을 통해 처리된다.
- 자격 증명이 저장소와 Docker 이미지에 포함되지 않는다.
- DB 백업과 복구 절차가 검증되어 있다.
- 이전 NestJS 이미지로 롤백하는 절차가 검증되어 있다.

## 13. 당장 진행할 작업

1. [x] 신규 `iwtc-backend-nest` 저장소 생성
2. [x] Cursor workspace에 신규 저장소 추가
3. [x] 기존 프론트엔드 기준 API 호출 목록 작성
4. [ ] 기존 Spring 코드에서 핵심 비즈니스 규칙 추출
5. [ ] Host Docker PostgreSQL 18과 API 도메인 구성 확정
6. [ ] K3s에서 Host PostgreSQL로 연결되는 Service와 EndpointSlice 설계
7. [x] NestJS 12 프로젝트와 Prisma 7.10 기본 구조 생성
8. [ ] 최초 Prisma Schema, migration, seed 작성 및 실제 PostgreSQL 적용 확인
9. [ ] Dockerfile과 로컬 Compose 구성 및 이미지 빌드 확인
10. [ ] GitHub Actions, GHCR, Kubernetes, Argo CD PreSync 기본 배포 구성
11. [ ] PostgreSQL 외부 백업과 restore test 구성
12. [x] 월드컵 공개 목록 API 구현과 계약 테스트

8번과 9번의 파일 구성은 완료했다. 현재 로컬 Docker 데몬이 꺼져 있어 실제 PostgreSQL migration·seed 적용과 Docker 이미지 빌드 확인만 남아 있다.
