# SSO Sidecar (nginx-hash-lock) - 전체 문서

## 개요

nginx-hash-lock은 모든 AppStore 앱 앞에 붙는 SSO 프록시 사이드카 컨테이너다.
앱 컨테이너 자체를 수정하지 않고, 환경변수만으로 SSO + 자동 로그인을 처리한다.

## 아키텍처

```
브라우저
  │
  ▼
Caddy (리버스 프록시)
  │
  ▼
┌─────────────────────────────┐
│  nginx-hash-lock (sidecar)  │
│  ├─ nginx (port 80)         │  ← 외부 요청 수신
│  │   └─ auth_request ───────┼──→ auth-service (port 9999)
│  │                          │       ├─ 세션 관리
│  │                          │       ├─ OIDC (Authelia 연동)
│  │                          │       └─ auto-login
│  └─ proxy_pass ─────────────┼──→ 앱 컨테이너 (backend)
└─────────────────────────────┘
```

## 인증 흐름

### 1. 최초 접속 (세션 없음)
```
브라우저 → sidecar → auth_request → 401 (세션 없음)
                   → Authelia SSO 로그인 redirect
                   → 사용자가 SSO 로그인
                   → OIDC 콜백 → nginxhashlock_session 쿠키 생성
                   → auto-login (앱 로그인 API 호출)
                   → 앱 대시보드 바로 진입
```

### 2. 재방문 (세션 있음)
```
브라우저 → sidecar → auth_request → 200 (세션 유효)
                   → proxy_pass → 앱 컨테이너
```

## 환경변수 (docker-compose에서 설정)

### 필수

| 환경변수 | 설명 | 예시 |
|----------|------|------|
| `OIDC_REGISTRAR_URL` | Authelia OIDC 등록 서비스 URL | `http://auth-registrar:9092` |
| `BACKEND_HOST` | 앱 컨테이너 hostname | `filebrowser-app` |
| `BACKEND_PORT` | 앱 컨테이너 포트 | `80` |
| `LISTEN_PORT` | sidecar 리스닝 포트 | `80` |

### Auto-login (앱 자체 로그인 자동 처리)

| 환경변수 | 설명 | 예시 |
|----------|------|------|
| `AUTO_LOGIN_URL` | 앱의 로그인 API 경로 | `/api/login` |
| `AUTO_LOGIN_BODY` | 로그인 요청 body | `{"username":"admin","password":"xxx"}` |
| `AUTO_LOGIN_METHOD` | HTTP 메서드 (기본: POST) | `POST` |
| `AUTO_LOGIN_CONTENT_TYPE` | Content-Type (기본: application/json) | `application/json` |
| `AUTO_LOGIN_COOKIE_NAME` | response body를 쿠키로 설정할 이름 | `auth` |
| `AUTO_LOGIN_LOCALSTORAGE_KEY` | response body를 localStorage에 설정할 키 | `jwt` |

### 기타

| 환경변수 | 설명 | 예시 |
|----------|------|------|
| `PROXY_AUTH_USERNAME` | X-Auth-User 헤더에 고정 username 설정 | `admin` |
| `SESSION_DURATION_HOURS` | 세션 유효 기간 (기본: 720시간 = 30일) | `720` |

## 앱별 인증 패턴

### 패턴 1: Set-Cookie 방식 (가장 간단)
앱이 로그인 API 응답에 `Set-Cookie` 헤더로 세션 쿠키를 줌.

```yaml
environment:
  AUTO_LOGIN_URL: "/login"
  AUTO_LOGIN_BODY: '{"username":"admin","password":"$APP_DEFAULT_PASSWORD"}'
```

sidecar가 Set-Cookie를 그대로 브라우저에 전달. 추가 설정 불필요.

**해당 앱**: 대부분의 앱 (qBittorrent, Nextcloud 등)

### 패턴 2: Response Body 토큰 방식
앱이 로그인 API 응답 body에 토큰(JWT 등)을 반환.

```yaml
environment:
  AUTO_LOGIN_URL: "/api/login"
  AUTO_LOGIN_BODY: '{"username":"admin","password":"$APP_DEFAULT_PASSWORD"}'
  AUTO_LOGIN_COOKIE_NAME: "auth"
```

sidecar가 body를 읽어서 `auth` 쿠키로 설정.

### 패턴 3: Response Body + localStorage 방식
앱 프론트엔드가 쿠키가 아닌 localStorage에서 토큰을 읽음.

```yaml
environment:
  AUTO_LOGIN_URL: "/api/login"
  AUTO_LOGIN_BODY: '{"username":"admin","password":"$APP_DEFAULT_PASSWORD"}'
  AUTO_LOGIN_COOKIE_NAME: "auth"
  AUTO_LOGIN_LOCALSTORAGE_KEY: "jwt"
```

sidecar가 redirect 대신 HTML 페이지를 보내서 `localStorage.setItem('jwt', token)` 실행 후 redirect.

**해당 앱**: FileBrowser (v2.x)

### 패턴 4: Proxy Auth 방식
앱이 `Remote-User` 헤더를 신뢰해서 자동 로그인.

```yaml
environment:
  PROXY_AUTH_USERNAME: "admin"
```

앱 자체에서 `--auth.method=proxy` 설정 필요 (pre-install-cmd에서 처리).
auto-login 환경변수 불필요.

## docker-compose 예시 (FileBrowser)

```yaml
name: filebrowser
services:
  filebrowser-app:
    container_name: filebrowser-app
    image: filebrowser/filebrowser:v2.63.1
    environment:
      FB_DATABASE: /db/database.db
    volumes:
      - /DATA/AppData/$AppID/db/:/db/
      - /DATA/:/srv
    expose:
      - 80
    networks:
      - pcs

  filebrowser:
    image: ghcr.io/bookjjun-ij/nginx-hash-lock:main
    container_name: filebrowser
    hostname: filebrowser
    environment:
      OIDC_REGISTRAR_URL: "http://auth-registrar:9092"
      BACKEND_HOST: "filebrowser-app"
      BACKEND_PORT: "80"
      LISTEN_PORT: "80"
      AUTO_LOGIN_URL: "/api/login"
      AUTO_LOGIN_BODY: '{"username":"admin","password":"$APP_DEFAULT_PASSWORD"}'
      AUTO_LOGIN_COOKIE_NAME: "auth"
      AUTO_LOGIN_LOCALSTORAGE_KEY: "jwt"
    depends_on:
      - filebrowser-app
    labels:
      caddy_0: filebrowser-${APP_DOMAIN}
      caddy_0.import: gateway_tls
      caddy_0.reverse_proxy: "{{upstreams 80}}"
    networks:
      - pcs

x-casaos:
  pre-install-cmd: |
    docker run --rm -v /DATA/AppData/$AppID:/data alpine sh -c "mkdir -p /data/db && chown -R $PUID:$PGID /data"
    docker run --rm -v /DATA/AppData/$AppID/db:/db -e FB_DATABASE=/db/database.db filebrowser/filebrowser:v2.63.1 config init --address 0.0.0.0 --port 80 2>/dev/null || true
    docker run --rm -v /DATA/AppData/$AppID/db:/db -e FB_DATABASE=/db/database.db filebrowser/filebrowser:v2.63.1 users add admin "$APP_DEFAULT_PASSWORD" --perm.admin 2>/dev/null || true
    docker run --rm -v /DATA/AppData/$AppID:/data alpine sh -c "chown -R $PUID:$PGID /data"
```

## pre-install-cmd 가이드

pre-install-cmd는 앱 기본 초기화용. SSO 설정이 아님.

**필요한 경우:**
- 디렉토리 생성 + 권한 설정 (거의 모든 앱)
- DB 초기화 + admin 유저 생성 (비밀번호를 $APP_DEFAULT_PASSWORD로 맞추기 위해)

**불필요한 경우:**
- Set-Cookie 방식 앱에서 기본 비밀번호가 고정인 경우
- 앱이 자동으로 admin 유저를 생성하고, 그 비밀번호를 AUTO_LOGIN_BODY에 맞추면 됨

## 핵심 파일

### sidecar 내부

| 파일 | 역할 |
|------|------|
| `auth-service/app.js` | Node.js 인증 서비스 (OIDC, 세션, auto-login) |
| `nginx.conf` | nginx 설정 (auth_request, proxy_pass, Remote-User 헤더) |
| `entrypoint.sh` | 컨테이너 시작 스크립트 (nginx + auth-service 실행) |

### 로그 확인

```bash
# auth-service 로그 (auto-login 결과 등)
docker exec <sidecar-container> cat /var/log/auth-service.log | tail -20

# nginx 로그
docker exec <sidecar-container> cat /var/log/nginx/error.log | tail -20
```

## 빌드 & 배포

```bash
# 1. 코드 수정 후 커밋/push
cd Nginx-hash-lock
git add -A && git commit -m "변경 내용" && git push

# 2. GitHub Actions 빌드 트리거
gh workflow run docker-publish.yml --repo BookJJun-IJ/Nginx-hash-lock

# 3. 서버에서 새 이미지 적용
docker pull ghcr.io/bookjjun-ij/nginx-hash-lock:main
docker restart <sidecar-container>
```

## 디버깅 체크리스트

1. **SSO 로그인은 되는데 앱 자체 로그인이 나옴**
   - auth-service 로그 확인: auto-login 성공 여부
   - 403이면: 비밀번호 불일치 → DB 재생성 또는 AUTO_LOGIN_BODY 수정
   - 성공(200)인데 로그인 나옴: 쿠키/localStorage 문제 → AUTO_LOGIN_COOKIE_NAME, AUTO_LOGIN_LOCALSTORAGE_KEY 확인

2. **Bad Gateway**
   - BACKEND_HOST, BACKEND_PORT 확인
   - 앱 컨테이너가 0.0.0.0에서 리스닝하는지 확인
   - `config init --address 0.0.0.0` 필요할 수 있음

3. **OIDC 콜백 에러**
   - auth-registrar 컨테이너가 실행 중인지 확인
   - `docker exec <sidecar> cat /var/log/auth-service.log` 에서 에러 확인

4. **앱이 AppStore에 안 보임**
   - docker-compose YAML 문법 에러 → `docker-compose config` 로 검증
   - YAML folding (`>`) 주의: CasaOS 파서가 까다로움, list 형식 추천

## 현재 상태 (2026-06-10)

- sidecar 이미지: `ghcr.io/bookjjun-ij/nginx-hash-lock:main`
- 지원 기능: OIDC SSO, auto-login (Set-Cookie / body→cookie / body→localStorage)
- 테스트 앱: FileBrowser (패턴 3: body + localStorage)
- 미완료: 서버에서 새 이미지 pull + FileBrowser 재설치 후 end-to-end 테스트

## FileBrowser 특이사항

- 로그인 API: `POST /api/login` → JWT를 response body로 반환 (Set-Cookie 아님)
- 프론트엔드: `localStorage.jwt`에서 토큰 읽음 (쿠키 `auth`는 서버 요청용)
- `users update --password` CLI 명령이 작동하지 않음 → `users rm` + `users add` 사용
- `config init` 기본값이 `address: 127.0.0.1` → 반드시 `--address 0.0.0.0` 지정
- Pinia store 사용, `isLoggedIn`은 `state.user !== null`로 판단
- `validateLogin()`이 `localStorage.getItem("jwt")`를 읽어서 `/api/renew` 호출
