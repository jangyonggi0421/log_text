# Rocky Linux 9.7 SSH 설정

```bash

# 1. 패키지 업데이트
sudo dnf update -y
sudo dnf upgrade -y

# 2. ssh 상태 확인 / 설치 확인
sudo systemctl status sshd
rpm -qa | grep openssh-server

# 3. ssh 가 시작 중이지 않을 경우:
sudo systemctl start sshd

## 3-1. ssh가 없을 경우:
sudo dnf install openssh-server

## 3-2. 시스템 부팅 시 자동 실행 등록 및 서비스 활성화, 즉시 시작
sudo systemctl enable --now sshd


# 4. 방화벽 설정:
sudo firewall-cmd --list-all

## 4-1. services 에서 ssh 가 없을 경우:
sudo firewall-cmd --permanent --add-services=ssh

## 4-2. 반영
sudo firewall-cmd --reload


# 5. .ssh 디렉터리 생성 및 소유자 변경, 권한 설정
sudo mkdir .ssh
sudo chown user:user .ssh/
sudo chmod 700 .ssh/


# 6. authorized_keys 파일이 없을 경우 생성
sudo nano .ssh/authorized_keys
# ctrl+O 하여 [엔터]로 저장 후 ctrl+X 로 나가기


# 7. SELinux 보안 컨텍스트(라벨)를 기본값으로 복구
restorecon -Rv ~/.ssh/


# 8. ssh 설정 변경
sudo vi /etc/ssh/sshd_config

# 수정 사항 (주석 해제 및 설정 확인):
# 45라인: PubkeyAuthentication yes # 인증서 로그인 설정
# 49라인: AuthorizedKeyFile .ssh/authorized_keys  # 인증키 파일 설정
# 65라인: PasswordAuthentication yes (키 등록 전까지 한시적 허용) # 비밀번호 로그인 설정(인증키 로그인 설정 후 no로 변경)
# PermitRootLogin no    # Root 로그인 불가
# 이후 esc -> :wq 하여 저장


# 8-1. 22번 포트 변경 시
## 8-2. sshd_config 파일에서 Port=22 인 곳을 찾아 주석을 해제하고
## 8-3. SELinux에 xxx번 포트를 SSH 서비스 포트로 등록
sudo semanage port -a -t ssh_port_t -p tcp xxx


# 9. sshd 재시작
sudo systemctl restart sshd


# 10. 인증서 로그인 설정

## 10-1. ssh 키 생성
ssh-keygen -t rsa -b 4096
# -t : 키 타입(RSA 암호화), -C : comment(생락가능), -b : 4096 비트 길이
# Enter file in which to save thse key (/root/.ssh/id_rsa) : 파일명 입력(공백 처리 가능 : Enter)
# Enter passphrase for */root/.ssh/id_rsa* (empty for no passphrase) : 암호 입력(공백 처리 가능 : Enter)
# Enter same passphrase again : 암호 재입력(공백 처리 가능 : Enter)

## 10-2. (Windows) 생성된 공개키(.pub) 내용 클립보드 복사
Get-Content $env:C:\Users\<사용자명>\.ssh\<키 이름>.pub | Set-Clipboard

## 10-2. (macOS) 공개키 등록
ssh-copy-id user@192.168.0.1


## 11. ssh 접속
ssh user@192.168.0.1







```
