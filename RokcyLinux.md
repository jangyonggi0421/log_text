# Rocky Linux 설정 모음

```bash

# 업데이트:
sudo dnf update -y
sudo dnf upgrade -y


# root 계정 변경:
sudo passwd root

# 계정 추가(adduser):
sudo adduser [사용자ID]

# 비밀번호 설정(passwd):
sudo passwd [사용자ID]


# 관리자 권한(sudo) 부여: 새로 만든 계정에 관리자 권한을 주려면 wheel 그룹(Rocky Linux, RHEL 계열)이나 sudo 그룹(Ubuntu 계열)에 추가
sudo usermod -aG wheel [사용자ID]


# 서비스 종료:
shutdown -h now


# 현재 실행 중인 커널 확인
uname -r

## 1. 시스템에 설치된 모든 커널 목록 확인
rpm -q kernel

## 2. 구버전 커널 삭제 (GRUB 자동 반영)
sudo dnf remove kernel-core-5.14.0-503.21.1.el9_5.x86_64
# 주의: kernel 패키지만 지우면 의존성 파일이 남을 수 있으므로, 핵심 패키지인 kernel-core-[버전]을 지정하여 지우는 것이 가장 깔끔. 나머지 의존성 패키지도 함께 삭제.

#------------------------------
## 추가 팁: 향후 커널 자동 관리 설정
#시스템에 남겨둘 커널의 최대 개수를 지정할 수 있음.
# 1. /etc/dnf/dnf.conf 파일을 열고
# sudo nano /etc/dnf/dnf.conf
# installonly_limit=3 항목을 찾아서 원하는 개수(예: 최신 2개만 남기려면 2)로 수정 후 저장. 업데이트 시 오래된 커널이 자동으로 삭제.
# -------------------------------



```