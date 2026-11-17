# infra

같은 Lightsail 서버에서 도는 앱들의 공용 Caddy(리버스 프록시, HTTPS 인증서). 결정 배경은 [`docs/adr/0002`](docs/adr/0002-caddy-in-separate-infra-repo.md).

| 주소 | 넘기는 곳 | 저장소 |
|---|---|---|
| `{$DOMAIN}` | `app:8080`, `/java-quiz/*`는 `~/app/static` | `b-now-29/app` |
| `stadium.{$DOMAIN}` | `stadium:8080` (WebSocket 포함), `/privacy`는 이 저장소의 `static/stadium/privacy.html` | `b-now-29/zipgwan` |

- 앱 compose는 각자의 서비스를 외부 Docker 네트워크 `edge`에 붙인다. Caddy는 서비스 이름으로 찾는다. DB는 `edge`에 붙이지 않는다.
- 인증서는 `app_caddy_data`·`app_caddy_config` volume에 있다(예전 `app` compose가 만든 이름 그대로). 지우면 재발급되고 그동안 HTTPS가 멈춘다.

## 서버 준비 (한 번)

```bash
mkdir -p ~/infra
echo 'DOMAIN=shwan.be' > ~/infra/.env      # 실제 값. 저장소에는 .env.example만 있다
docker network create edge || true
```

GitHub Secrets: `DEPLOY_SSH_KEY`(서버 SSH 개인 키), `DEPLOY_HOST`(서버 주소). `b-now-29/app`과 같은 값이다.

## 배포

`main`에 push하면 workflow가 `Caddyfile` 문법을 검사하고, `compose.yaml`과 `Caddyfile`을 `~/infra/`에 복사한 뒤 `caddy reload`한다. 새 Caddy가 아직 떠 있지 않으면(전환 전) 복사만 한다.

## 처음 전환 (한 번, 한가한 시간에, 수 초 중단)

예전에는 Caddy가 `b-now-29/app`의 compose 안에 있었다. `compose up -d`는 compose 파일에서 사라진 서비스의 컨테이너를 지우지 않으므로 한 번 손으로 바꾼다.

1. `docker network create edge || true`, 그리고 되돌릴 때 쓸 예전 설정을 남겨 둔다: `cp ~/app/compose.yaml ~/app/compose.caddy-backup.yaml`
2. `b-now-29/app`의 Caddy 제거 변경을 배포한다(`app`이 `edge`에 붙는다). 기존 `app-caddy-1`은 계속 돈다.
3. 이 저장소의 `main`을 배포해 `~/infra/`에 파일을 둔다(새 Caddy는 아직 안 뜬다).
4. 전환:
   ```bash
   docker stop app-caddy-1 && docker rm app-caddy-1
   cd ~/infra && docker compose up -d
   ```
5. 확인:
   ```bash
   curl -sI https://shwan.be | head -1
   curl -sI https://shwan.be/java-quiz/ | head -1
   docker compose logs caddy | grep -iE 'obtain|certificate' | head   # 기존 도메인 인증서를 새로 받지 않았는지
   ```

되돌리기(새 Caddy가 이상할 때):
```bash
cd ~/infra && docker compose down
cd ~/app && docker compose -f compose.caddy-backup.yaml up -d --no-deps caddy   # 예전 Caddy만(같은 인증서 volume, ~/app/Caddyfile)
```
