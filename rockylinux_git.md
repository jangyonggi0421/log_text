# git 설치(Rocky Linux 9.7)

```bash

# 1. 패키지 업데이트
sudo dnf update -y

# 2. git 다운로드
sudo dnf install git -y

# 3. 설치 확인
git --version

# 4. git 계정 설정
## 4-1. 사용자 이름 설정
git config --global user.name "본인의 닉네임"

## 4-2. 이메일 주소 설정
git config --global user.email "GitHub 가입 이메일"

## 4-3. 설정 확인
git config --list




```