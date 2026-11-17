# Caddy는 별도 인프라 저장소에서 관리하고 앱들은 공용 네트워크 `edge`로 붙는다

> `b-now-29/zipgwan`의 `docs/adr/0002-caddy-in-separate-infra-repo.md`에서 복사했다. 결정의 원본은 그쪽이다.

스타디움 서버는 기존 앱(`b-now-29/app`)과 같은 Lightsail 서버에 `stadium.<도메인>` 서브도메인으로 배포한다. 원래 Caddy는 기존 앱의 compose에 들어 있었고, 그 앱을 배포할 때마다 Caddyfile을 덮어썼다. 그래서 다른 앱의 라우팅이 한 앱 저장소에 묶이지 않도록 Caddy를 별도 인프라 저장소로 분리했다. 각 앱 compose는 서버에 한 번 만들어 둔 외부 Docker 네트워크 `edge`에 붙고, Caddy는 서비스 이름(`app:8080`, `stadium:8080`)으로 요청을 넘긴다.

## Considered Options

- **기존 `app` 저장소의 Caddyfile에 스타디움 블록 추가**: 가장 적게 바뀌지만, 스타디움 라우팅을 바꾸려면 관련 없는 저장소를 고쳐 배포해야 한다. 기각했다.
- **jar + systemd로 서버에서 직접 실행**: Caddy가 컨테이너 안에 있어서 `localhost`로는 서버 본체의 앱에 닿지 않는다. 기각했다.

## Consequences

- 인증서는 기존 volume(`app_caddy_data`, `app_caddy_config`)을 외부 volume으로 이어 쓴다. 새로 시작하면 기존 도메인 인증서가 재발급되고, 그동안 HTTPS가 멈춘다.
- `edge` 네트워크는 각 배포 스크립트에서 `docker network create edge || true`로 없을 때만 만든다. prune이나 서버 재구축으로 사라져도 복구된다.
- 기존 Caddy에서 새 Caddy로 넘어갈 때는 `compose up -d`가 compose 파일에서 사라진 서비스의 컨테이너를 지우지 않으므로, 한 번 손으로 전환한다. 이때 수 초간 중단된다.
- `/java-quiz` 정적 파일은 서버의 `~/app/static`을 그대로 마운트한다.
- `postgres`는 `edge`에 붙이지 않는다.
