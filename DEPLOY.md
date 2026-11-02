# 배포 가이드 - 시놀로지 나스

## 데이터베이스 설정

### PostgreSQL 활성화
이 프로젝트는 현재 SQLite를 기본으로 사용하지만, docker-compose에 PostgreSQL 서비스가 포함되어 있습니다.

#### 1. .env 파일 설정
```bash
# PostgreSQL 자격증명 설정 (원래 비밀번호로 변경)
POSTGRES_DB=guminai_db
POSTGRES_USER=guminai_user
POSTGRES_PASSWORD=your_actual_secure_password_here
DATABASE_URL=postgresql://guminai_user:your_actual_secure_password_here@database:5432/guminai_db
```

#### 2. 볼륨 경로 확인
- 시놀로지 나스 PostgreSQL 데이터: `/volume1/docker/guminai/db_data`

#### 3. PostgreSQL 컨테이너 확인
```bash
docker-compose ps
# 상태: guminai-db - healthy
```
`.env.template`를 기반으로 `.env` 파일을 생성하고 실제 값으로 채웁니다:

```bash
cp .env.template .env
# .env 파일 편집: CLOVA_API_KEY, CLOVA_PRIMARY_KEY, CLOVA_REQUEST_ID, SECRET_KEY 등 설정
```

### 2. 시놀로지 나스 경로 확인
- 코드 디렉토리: `/volume1/docker/guminai`
- docker-compose.yml에서 마운트 포인트가 맞는지 확인

### 3. Docker 빌드 및 실행

#### 초기 빌드 및 실행
```bash
cd /volume1/docker/guminai
docker-compose up -d --build
```

#### 로그 확인
```bash
docker-compose logs -f app
```

## 운영 관리

### 배포 (무중단 워커 재로드)

1. **코드 변경 사항 확인**
```bash
docker-compose down
# 코드 수정
```

2. **이미지 재빌드 및 재실행**
```bash
docker-compose up -d --build
```

3. **또는 그레이스풀 리로드 (워커 재시작만)**
```bash
touch /tmp/guminai.reload
```

### 환경 변수 갱신 (ASSET_VERSION 등)

```bash
# .env 파일 수정
nano .env
# ASSET_VERSION=1234567890 (현재 타임스탬프로 설정)

# 컨테이너 재시작
docker-compose restart app
```

### 상태 확인

```bash
# 컨테이너 상태
docker-compose ps

# 운영 로그
docker-compose logs app --tail 100

# 응답 헤더 확인 (캐시 정책 검증)
curl -I http://localhost:8000/community
curl -I http://localhost:8000/static/css/styles.css
```

## 캐시 정책 검증

### HTML/API 응답 (no-store 확인)
```bash
curl -I http://localhost:8000/community
# 예상: Cache-Control: no-store, no-cache, must-revalidate, max-age=0
```

### 정적 파일 응답 (장기 캐시 확인)
```bash
curl -I http://localhost:8000/static/css/styles.css?v=<asset_version>
# 예상: Cache-Control: public, max-age=31536000, immutable
```

## 트러블슈팅

### 포트 충돌
- docker-compose.yml에서 포트 매핑 변경: `ports: ["9000:8000"]`

### uWSGI 소켓 권한 오류
- uwsgi.ini에서 `uid=1029, gid=100` 확인 (시놀로지 서비스 계정)

### 정적 파일 404
- 볼륨 마운트 경로 재확인: `/volume1/docker/guminai:/app`

### 템플릿 변경이 반영 안 됨
- ASSET_VERSION을 새 값으로 갱신하고 컨테이너 재시작
- 브라우저 캐시 클리어 (Ctrl+Shift+Del 또는 시크릿 창)

## 보안 체크리스트

- [ ] `.env` 파일이 `.gitignore`에 포함되어 있는가
- [ ] `SECRET_KEY`가 강력한 무작위 문자열로 설정되어 있는가
- [ ] `CLOVA_API_KEY` 등이 로그에 노출되지 않는가 (docker-compose logs 확인)
- [ ] uWSGI가 root가 아닌 서비스 계정(1029)으로 실행되는가
- [ ] 시놀로지 나스 파이어월에서 포트 8000 허용 설정 확인

## 모니터링

### 헬스체크 설정
docker-compose.yml의 healthcheck가 자동으로 30초마다 `/healthz` 엔드포인트를 확인합니다.
컨테이너가 10회 연속 실패하면 unhealthy로 표시됩니다.

### 재시작 정책
`restart: unless-stopped` - 실패 시 자동 재시작, 명시적 중지는 유지
