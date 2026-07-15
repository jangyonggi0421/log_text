# RockyLinux 원격 데스크톱 설정

## cockpit 설정

```bash

# 1. cockpit 설치
sudo dnf install -y cockpit

# 2. 부팅 시 자동 사작하게 설정 및 즉시 실행
sudo systemctl enable --now cockpit.socket

# 3. 방화벽 설정
sudo firewall-cmd --permanent --add-service=cockpit 
sudo firewall-cmd --reload

```

---

## XRDP 설정

```bash

# 1. epel 저장소 추가
sudo dnf install epel-release -y


# 2. XRDP 설치
sudo dnf install xrdp -y


# 3. 부팅 시 자동 사작하게 설정 및 즉시 실행
sudo systemctl enable --now xrdp


# 4. 방화벽 설정
sudo firewall-cmd --permanent --add-port=3389/tcp
sudo firewall-cmd --reload



```