
# Dealmungchi NGINX 설정 템플릿

- [nginx-certbot](https://github.com/wmnnd/nginx-certbot)
- [nginx-badbot-blocker](https://github.com/mariusv/nginx-badbot-blocker)
- [nginx-ultimate-bad-bot-blocker](https://github.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker)

## 주요 기능

- **SSL 인증서 자동화**: `init-letsencrypt.sh` 스크립트로 ssl 인증서를 자동으로 발급합니다.
- **Bad Bot Block**: `nginx-badbot-blocker`와 `nginx-ultimate-bad-bot-blocker`로 악성봇을 차단 합니다.

## 설치 및 설정

### 1. Docker Compose 설정 예시

```yaml
services:
  nginx:
    image: nginx:1.15-alpine
    restart: unless-stopped
    volumes:
      - ./data/nginx/conf.d:/etc/nginx/conf.d
      - ./data/nginx/bots.d:/etc/nginx/bots.d
      - ./data/nginx/sites-available:/etc/nginx/sites-available
      - ./data/nginx/sites-enabled:/etc/nginx/sites-enabled
      - ./data/nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./data/certbot/conf:/etc/letsencrypt
      - ./data/certbot/www:/var/www/certbot
      - /usr/local/share/data:/usr/share/nginx/html/thumbnails
    ports:
      - "80:80"
      - "443:443"
    command: |
      /bin/sh -c '
        apk update &&
        apk add --no-cache curl bind-tools busybox-extras mailx ssmtp openssl &&
        mkdir -p /usr/local/sbin &&
        wget https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/refs/heads/master/install-ngxblocker -O /usr/local/sbin/install-ngxblocker &&
        wget https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/refs/heads/master/update-ngxblocker -O /usr/local/sbin/update-ngxblocker &&
        chmod +x /usr/local/sbin/install-ngxblocker &&
        chmod +x /usr/local/sbin/update-ngxblocker &&
        echo "00 */8 * * * /usr/local/sbin/update-ngxblocker -e sjsage522.dev@gmail.com" > /etc/crontabs/root &&
        crond &&
        while :; do
          sleep 6h &
          wait $${!};
          nginx -s reload;
        done &
        nginx -g "daemon off;"
      '

  certbot:
    image: certbot/certbot
    restart: unless-stopped
    volumes:
      - ./data/certbot/conf:/etc/letsencrypt
      - ./data/certbot/www:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
```

### 2. 인증서 최초 발급

- `init-letsencrypt.sh` 스크립트를 사용하여 **Let's Encrypt**의 SSL 인증서를 최초 발급합니다.

### 3. NGINX 가상 호스트 설정

- 가상 호스트 파일은 `.vhost` 확장자를 사용해야 하며, `sites-available` 디렉토리에 설정을 작성하고, 이를 `sites-enabled`에 심볼릭 링크로 연결하여 관리합니다.
- `conf.d` 디렉토리에는 NGINX 설정에 필요한 커스텀 설정 파일을 배치합니다.

#### 예시:
`/etc/nginx/sites-available/yourdomain.vhost`:

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    # 인증서 발급을 위한 경로
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
}

# init-letsencrypt.sh 스크립트로 인증서 최초 발급시에는 주석 처리 해야함.
server {
    listen 443 ssl;
    server_name yourdomain.com www.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    # 사이트 설정
    location / {
        root /var/www/yourdomain;
        index index.html;
    }
}
```

`/etc/nginx/sites-enabled/`에 심볼릭 링크를 추가하여 활성화:

```bash
ln -s /etc/nginx/sites-available/yourdomain.vhost /etc/nginx/sites-enabled/
```

### 4. Bad Bot Block 설정

- `nginx-badbot-blocker`와 `nginx-ultimate-bad-bot-blocker`는 자동으로 악성 봇을 차단합니다.
- `nginx-ultimate-bad-bot-blocker`는 **자동 업데이트** 기능을 제공하며, 이 기능을 통해 최신 봇 목록을 자동으로 갱신합니다.
- 이 갱신 작업은 **cron job**을 사용하여 주기적으로 실행되며, 이메일을 통해 결과를 받을 수 있습니다.

| `nginx-ultimate-bad-bot-blocker` 최초 세팅은 https://github.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker 를 참고하였음

### 5. 이메일 발송 설정

이메일 발송을 위해 `ssmtp` , `mailx`, `openssl`와 같은 툴이 필요합니다.  `nginx-ultimate-bad-bot-blocker`의 자동 갱신 완료 알림을 이메일로 받을 수 있습니다.

#### ssmtp 설정

- `/etc/ssmtp/ssmtp.conf` 파일을 설정하여 Gmail SMTP 서버를 통해 이메일을 전송합니다.

```bash
root=your_email@gmail.com
mailhub=smtp.gmail.com:587
AuthUser=your_email@gmail.com
AuthPass=your_app_password  # must use application password
UseTLS=YES
UseSTARTTLS=YES
FromLineOverride=YES
hostname=localhost
```

### 6. 설정 규칙

- **가상 호스트 파일**: `.vhost` 확장자를 사용하여 관리합니다.
  - `sites-available`에 설정 파일을 작성하고, `sites-enabled`에 심볼릭 링크로 연결합니다.
  - `conf.d`는 커스텀 설정을 위한 디렉토리로 사용합니다.
- **Bad Bot Block 설정**:  
  - `nginx-badbot-blocker`
    - conf.d/block.conf
    - conf.d/nginx-badbot-blocker

  - `nginx-ultimate-bad-bot-blocker`
    - conf.d/botblocker-nginx-settings.conf
    - conf.d/globalblocklist.conf
    - .vhost 파일 내 include로 참조



