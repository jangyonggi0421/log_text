# 서버 구성

## 1. Apache: rpm 다운로드 후 설치(예: 페쇄망)

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

---

## 2. OpenJDK 21 설치

1. 파일 다운로드 = `https://jdk.java.net/archive/`

2. 파일 압축해체

```bash
sudo tar -xvf openjdk-21_linux-x64_bin.tar.gz -C /opt/

```
