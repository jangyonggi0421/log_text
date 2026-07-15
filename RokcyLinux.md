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









```