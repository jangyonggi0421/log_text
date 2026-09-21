# Apache 웹서버 설치

## rpm 다운로드 후 설치(예: 페쇄망)

1. 파일 다운로드

```bash
mkdir ~/httpd-offline
cd ~/httpd-offline

sudo dnf download --resolve httpd
# 또는 더 명시적으로:
# sudo dnf download --resolve httpd mod_ssl
```

2. 설치

```bash
sudo dnf install -y *.rpm
```

3. 서비스 시작

```bash
sudo systemctl enable --now httpd
```
