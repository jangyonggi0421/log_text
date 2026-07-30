# VS Code(Linux 기반) 설치

## RHEL, Fedora 및 CentOS 기반 배포판
* 운영체제 : Rocky Linux 9 (커널 버전: 5.14.0-427.5.1.el9_4.x86_64)
​
1. Microsoft GPG 키 및 저장소 등록
```
# 1. GPG 키 가져오기
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc

# 2. VS Code 저장소 추가
sudo sh -c 'echo -e "[code]\nname=Visual Studio Code\nbaseurl=https://packages.microsoft.com/yumrepos/vscode\nenabled=1\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/vscode.repo'
```
<br>

2. VS Code 설치
```
# 1. 캐시 업데이트
sudo dnf check-update

# 2. VS code 설치
sudo dnf install code -y
```
<br>

3. code 명령어로 VS code 실행