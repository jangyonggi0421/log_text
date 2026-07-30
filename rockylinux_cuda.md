# Rocky Linux CUDA/nvidia 드라이버 설치

## Windows / Linux 멀티부팅 등 사용 시(Gemini)
* Secure Boot(안전 부팅)를 끄면 일어나는 변화는 크게 **NVIDIA 드라이버 측면**, **보안 측면**, 그리고 **Windows 멀티부팅 측면**

### 1. NVIDIA 드라이버가 "즉시" 작동
Secure Boot가 꺼지면 커널이 드라이버의 "서명(Signature)" 여부를 더 이상 검사하지 않습니다.
* **복잡한 MOK 서명 작업(`sign-file`, `xz` 압축 풀기 등)을 아예 안 해도 됩니다.**
* 드라이버를 새로 설치하거나 커널이 업데이트되어 드라이버가 재빌드되어도, 아무런 추가 작업 없이 **바로 `nvidia-smi`가 정상 작동**합니다.

---

### 2. 부팅 단계의 보안 수준이 살짝 낮아짐

Secure Boot는 컴퓨터가 켜질 때 악성코드(루트킷 등)가 운영체제보다 먼저 실행되어 시스템을 장악하는 것을 막아주는 하드웨어 보안 기술입니다.

* **보안적 영향:** 이 기능을 끄면 신뢰할 수 없는 부트로더나 서명되지 않은 커널 모듈이 실행될 수 있는 여지가 열립니다.
* **실제 체감 위험도:** 해커가 내 노트북을 물리적으로 탈취해서 변조하거나, 시스템 루트 권한을 해킹당하지 않는 이상 일반적인 웹 서핑이나 개발 환경에서 **컴퓨터가 바이러스에 쉽게 걸린다거나 하는 일은 거의 발생하지 않습니다.** 대부분의 리눅스 개발자들은 이 기능을 끄고 사용합니다.

---

### 3. Windows 멀티부팅 사용 시 주의사항 (BitLocker)

만약 갤럭시 북에 **Windows와 Rocky Linux를 멀티부팅**으로 같이 사용 중이시라면 아래 사항을 확인해야 합니다.

* **Windows 11 구동:** Windows 11은 원래 Secure Boot가 필수 사양이지만, 이미 설치된 상태라면 Secure Boot를 꺼도 정상적으로 부팅하고 사용하는 데 아무런 지장이 없습니다.
* **BitLocker(드라이브 암호화) 경고 ⚠️:** Windows에 **BitLocker**가 켜져 있는 상태에서 Secure Boot를 끄면, Windows로 부팅할 때 "하드웨어 변경이 감지되었습니다"라며 파란색 화면에 BitLocker 복구 키(48자리 숫자)를 입력하라고 뜹니다.
* *해결책:* Secure Boot를 끄기 전에 Windows로 부팅하여 BitLocker를 잠시 꺼두거나(보호 일시 중지), Microsoft 계정 페이지에서 48자리 복구 키를 미리 메모해 두어야 합니다. (BitLocker를 안 쓰신다면 상관없습니다.)

---

### 💡 요약 및 추천

* **추천 행동:** 갤럭시 북을 개인 작업/학습용으로 쓰신다면 **Secure Boot를 [Disabled]로 끄는 것을 강력히 추천**합니다.
* **이유:** 리눅스는 커널 업데이트가 잦은데, 그때마다 매번 이 복잡한 서명 작업을 반복하는 것은 엄청난 스트레스이기 때문입니다. 끄고 나면 드라이버 관련 삽질이 90% 이상 줄어듭니다!

---

## CUDA/nvidia 드라이버 설치

```bash

#### *******

# 





---

## 

---




---



## -----
# 1. 시스템 업데이트
sudo dnf upgrade --refresh -y


# 2. EPEL 및 CRB(CodeReady Builder) 저장소 활성화 (의존성 해결용)
sudo dnf config-manager --set-enabled crb
sudo dnf install epel-release -y


# 3. 커널 헤더 및 개발 도구 설치
sudo dnf install kernel-devel-$(uname -r) kernel-headers-$(uname -r) -y


# 4. 재부팅
sudo reboot


# 5. RPM Fusion 저장소 추가
sudo dnf install https://mirrors.rpmfusion.org/free/el/rpmfusion-free-release-9.noarch.rpm \
https://mirrors.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-9.noarch.rpm -y



# 6. NVIDIA 드라이버 설치
# 드라이버 및 CUDA 라이브러리 설치
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda -y


# 7. 커널 모듈 빌드 대기(3분 ~ 5분) 후 재부팅
modinfo nvidia
## 결과 화면에 드라이버 정보(filename, version: 580.159.04 등)가 정상적으로 출력되면 성공
## 재부팅
sudo reboot


# 8. 설치 확인
nvidia-smi
## 드라이버 정보, CUDA 정보 등이 뜨면 성공


#--------- 오류 ------------------
# NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver. Make sure that the latest NVIDIA driver is installed and running.

## ------ Secure Boot 사용 시(root 권한으로 수행) ---------
# 1. 드라이버 서명용 개인 키(MOK) 생성
# 서명 키를 보관할 디렉토리 생성 및 이동
sudo mkdir -p /root/mok
cd /root/mok

# 3650일(10년)짜리 자체 서명 키 생성
sudo openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -outform DER -out MOK.der -nodes -days 3650 -subj "/CN=Rocky Linux NVIDIA Driver MOK/"


# 2. 시스템(MOK 리스트)에 생성한 키 등록 요청
sudo mokutil --import MOK.der


# 3. 재부팅
sudo reboot


# 4. 재부팅 후 파란색 또는 회색 화면(Shim UEFI Key Management)이 나타나면 키보드 방향키와 Enter 키를 이용해 아래 순서대로 선택
# 4-1. 화면에 아무 키나 누르라는 메시지가 뜨면 아무 키나 누릅니다.
# 4-2. 메뉴가 나타나면 Enroll MOK를 선택합니다.
# 4-3. View key 0을 눌러 방금 만든 키(Rocky Linux NVIDIA Driver MOK)가 맞는지 확인한 뒤 (생략 가능), Continue를 선택합니다.
# 4-4. Yes를 선택하여 등록을 확정합니다.
# 4-5. 조금 전 단계 2에서 설정했던 임시 비밀번호를 입력합니다.
# 4-6. 마지막으로 Reboot을 선택하여 부팅을 진행합니다.


# 5. 커널 개발 패키지 정렬
## 1. 컴파일 환경 패키지 완벽 설치
dnf install gcc make elfutils-libelf-devel -y

## 2. 현재 실행 중인 커널 버전과 정확히 매칭되는 개발 도구 설치
dnf install kernel-devel-$(uname -r) kernel-headers-$(uname -r) -y
dnf upgrade kernel kernel-devel kernel-headers -y
reboot




# 5. 드라이버 패키지 강제 재설치(모듈 복구)
# root 계정인지 확인 (아니라면 sudo -i 먼저 실행)
## 1. dnf 캐시를 완전히 비웁니다.
dnf clean all

## 2. akmod-nvidia 패키지를 강제로 재설치합니다.
dnf reinstall akmod-nvidia -y


# 6. 커널 모듈 수동 빌드 명령 실행(3분 에서 5분 정도 소요)
akmods --force

# 5. 빌드된 NVIDIA 드라이버에 서명 적용하기
# 1. 작업 디렉토리 이동
cd /lib/modules/$(uname -r)/extra/nvidia-580xx/


# 2. 서명 진행
## 1. 사용할 경로 및 도구 정의
KERNEL_VERSION=$(uname -r)
TARGET_DIR="/lib/modules/$KERNEL_VERSION/extra/nvidia-580xx"
SIGN_TOOL="/usr/src/kernels/$KERNEL_VERSION/scripts/sign-file"

## 2. 안전하게 해당 디렉토리로 이동
cd $TARGET_DIR

## 3. [.ko.xz] 압축 풀기 -> [.ko] 파일로 변환 (이번엔 에러 없이 풀려야 합니다)
sudo xz -d *.ko.xz

## 4. 압축이 풀린 [.ko] 파일들에 서명 진행
for mod in *.ko; do
    echo "서명 중: $mod"
    sudo $SIGN_TOOL sha256 /root/mok/MOK.priv /root/mok/MOK.der "$mod"
done

## 5. 서명이 완료된 [.ko] 파일들을 다시 [.ko.xz]로 압축
sudo xz -z *.ko

## 6. 커널 모듈 의존성 데이터베이스 갱신
sudo depmod -a




# 3. 압축이 풀린 .ko 파일들에 서명 진행
KERNEL_VERSION=$(uname -r)
SIGN_TOOL="/usr/src/kernels/$KERNEL_VERSION/scripts/sign-file"

for mod in $(find . -name "*.ko"); do
    echo "Signing: $mod"
    sudo $SIGN_TOOL sha256 /root/mok/MOK.priv /root/mok/MOK.der "$mod"
done

# 4. 다시 xz로 압축 처리
sudo xz -z *.ko

# 5. 커널 모듈 의존성 데이터베이스 갱신
sudo depmod -a
## 현재 커널 버전 변수 지정
KERNEL_VERSION=$(uname -r)

## 커널 내부의 서명 도구 경로 지정
SIGN_TOOL="/usr/src/kernels/$KERNEL_VERSION/scripts/sign-file"

## NVIDIA 드라이버 파일들을 찾아 생성한 MOK 키로 서명 진행
for mod in $(find /lib/modules/$KERNEL_VERSION/extra/nvidia* -name "*.ko" -o -name "*.ko.xz"); do
    echo "Signing module: $mod"
    sudo $SIGN_TOOL sha256 /root/mok/MOK.priv /root/mok/MOK.der "$mod"
done


# 6. 드라이버 수동 로드 및 작동 확인

# 1. 패키지 업데이트
sudo dnf update -y


# 2. 현재 상태 확인
mokutil --sb-state
## SecureBoot enabled 이면


# 3. GPU 확인
lspci | grep -Ei "vga|3d|nvidia"


# 3. 필수 저장소 활성화
sudo dnf config-manager --set-enabled crb
sudo dnf install epel-release -y


# 4. 커널 개발 패키지 설치
sudo dnf install kernel-devel-matched kernel-headers -y


# 5. DKMS 빌드에 필요한 패키지 설치
sudo dnf groupinstall "Development Tools" -y
sudo dnf install dkms gcc make elfutils-libelf-devel -y
 

# 6. NVIDIA CUDA 저장소 등록
sudo dnf config-manager --add-repo \
https://developer.download.nvidia.com/compute/cuda/repos/rhel9/$(uname -i)/cuda-rhel9.repo
sudo dnf clean expire-cache


# 7. NVIDIA proprietary DKMS 모듈 활성화
sudo dnf module enable nvidia-driver:latest-dkms -y


# 8. 드라이버 설치
sudo dnf install cuda-drivers -y


# 9. DKMS 확인
dkms status
## nvidia/610.xx.xx, 5.14.0-687.25.1.el9_8.x86_64, x86_64: (installed 가 있어야 함)


# 10. MOK 등록
ls -l /var/lib/dkms/mok.*
## mok.pub가 있으면:
sudo mokutil --import /var/lib/dkms/mok.pub


# 11. 재부팅
sudo reboot


# 12. 부팅 후
mokutil --sb-state
## SecureBoot enabled 이면







# 6. NVIDIA 공식 저장소 추가
sudo dnf config-manager --add-repo \
https://developer.download.nvidia.com/compute/cuda/repos/rhel9/$(uname -i)/cuda-rhel9.repo


# 7. 캐시 갱신
sudo dnf clean expire-cache


# 8. 드라이버 설치
sudo dnf module enable nvidia-driver:latest-dkms -y



# 2. 설치
sudo dnf groupinstall "Development Tools" -y
### 커널 개발 도구 및 헤더 매칭 설치



# 3. ELRepo 저장소 등록
sudo rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org


# 4. 드라이버 설치
sudo dnf install https://www.elrepo.org/elrepo-release-9.el9.elrepo.noarch.rpm -y
sudo dnf install kmod-nvidia -y

# 5. 재부팅
sudo reboot


# 6. nvidia 드라이버 실행
nvidia-smi
## 실행결과: GPU 정보 및 드라이버 버전, CUDA 관련 정보가 뜨면 성공


# 7. 오류 시
## 7-1. 드라이버 모듈 수동 로드
sudo modprobe nvidia

## 7-2. 안되면
dpkg -l | grep nvidia-driver
## 또는 dpkg 꾸러미 설치 후 위 명령어 실행

## 7-3. 현재 설치 상태 확인 (RHEL 방식)
sudo dnf list installed | grep -i nvidia
### 커널 개발 도구 및 헤더 매칭 설치
### 현재 커널 버전에 정확히 일치하는 헤더 파일 설치 
sudo dnf install kernel-devel-$(uname -r) kernel-headers-$(uname -r)


## 7-4. EPEL 저장소 및 DKMS 설치
### 1. EPEL 저장소 활성화
sudo dnf install -y epel-release

### 2. 커널 개발 도구 패키지 설치 (드라이버 빌드용 필수)
sudo dnf install -y kernel-devel-$(uname -r) kernel-headers-$(uname -r) gcc make

### 3. dkms 설치
sudo dnf install -y dkms


## 7-5. 엔비디아 CUDA 저장소 추가 (RHEL 9 기준, 만약 로키 8버전이라면 rhel9 대신 rhel8 입력)
sudo dnf config-manager --add-repo https://nvidia.com


## 7-6. 엔비디아 드라이버 설치
sudo dnf install -y nvidia-driver nvidia-settings

# 출력된 정보(예: nvidia/550.54.14)를 바탕으로 빌드 진행
sudo dkms install nvidia/[확인된버전]580.159.04





코드를 사용할 때는 주의가 필요합니다.*만약 modprobe: ERROR: could not insert 'nvidia': Key was rejected by service 에러가 발생한다면, Secure Boot(안전 부팅)가 켜져 있어 차단된 것입니다. PC/서버 부팅 시 BIOS(UEFI) 설정에 진입하여 Secure Boot를 Disabled로 비활성화해 주세요.3. 드라이버가 없어 새로 설치해야 하는 경우Rocky Linux에서는 공식 엔비디아 저장소를 등록한 후 dnf로 간편하게 설치하는 것이 정석입니다.① 필수 의존성 및 EPEL 저장소 활성화bashsudo dnf install epel-release -y
# Rocky Linux 9 / 10인 경우 CRB 활성화
sudo dnf config-manager --set-enabled crb
# Rocky Linux 8인 경우 PowerTools 활성화
sudo dnf config-manager --set-enabled powertools
코드를 사용할 때는 주의가 필요합니다.② 시스템 환경에 맞는 개발 도구 설치bashsudo dnf install kernel-devel-$(uname -r) kernel-headers-$(uname -r) gcc make dkms -y
코드를 사용할 때는 주의가 필요합니다.③ 엔비디아 공식 CUDA 저장소 등록OS 버전에 맞는 레포지토리 파일을 NVIDIA Developer 다운로드 페이지를 참고하여 등록합니다.④ 드라이버 모듈 활성화 및 설치DKMS 기반의 최신 드라이버를 설치하여 커널 업데이트에 대응합니다.bashsudo dnf clean expire-cache




sudo dnf module enable nvidia-driver:latest-dkms -y
sudo dnf install nvidia-driver nvidia-settings -y
sudo dnf install cuda-driver -y
코드를 사용할 때는 주의가 필요합니다.⑤ 재부팅 후 확인bashsudo reboot
# 작동 확인
nvidia-smi
코드를 사용할 때는 주의가 필요합니다.







메타자료 내려받기에 실패하였습니다 (Error during downloading metadata) 에러는 DNF 패키지 매키저가 추가된 저장소(Repository) 주소에 접속하지 못했거나, 임시 캐시 파일이 꼬였을 때 발생합니다. [1, 2] 
이 문제를 해결하기 위해 다음 3단계 조치를 순서대로 진행해 보세요.
------------------------------
## 1단계: DNF 캐시 깨끗하게 비우기 (가장 흔한 해결책)
기존에 다운로드하다가 중단되었거나 잘못 저장된 메타데이터 캐시를 완전히 초기화합니다. [3] 

sudo dnf clean all
sudo rm -rf /var/cache/dnf
sudo dnf makecache


* 
* 결과 확인: 마지막 sudo dnf makecache 명령어가 에러 없이 정상적으로 저장소 목록을 불러오는지 확인합니다.
* 

------------------------------
## 2단계: 네트워크 및 인터넷 연결 상태 확인
간혹 리눅스의 네트워크 카드가 비활성화되어 있거나 DNS 설정 문제로 외부 저장소 서버에 접속하지 못할 수 있습니다. [2] 

# 구글 DNS 서버로 핑(Ping)을 보내 인터넷 연결 확인
ping -c 3 8.8.8.8


* 
* 만약 Network is unreachable 같은 메시지가 뜬다면?
서버의 네트워크 연결이 끊긴 상태입니다. 아래 명령어로 네트워크 장치를 켜주세요.

sudo nmcli connection up [본인_네트워크_장치명]# 또는 단순 네트워크 서비스 재시작
sudo systemctl restart NetworkManager

* 

------------------------------
## 3단계: 잘못 등록된 엔비디아 저장소 정리 및 재설정
버전이 맞지 않거나 잘못된 엔비디아 저장소 주소가 등록된 경우입니다. 기존 저장소 파일을 삭제하고 올바른 주소로 다시 추가합니다. [4] 

   1. 기존 잘못된 저장소 삭제
   
   sudo rm -f /etc/yum.repos.d/cuda-rhel*.repo
   
   2. Rocky Linux 버전에 맞는 저장소 재등록
   공식 [엔비디아 저장소](https://developer.download.nvidia.com/compute/cuda/repos/)에서 RHEL 8 또는 9용 레포지토리를 다시 추가합니다.

------------------------------
저장소 재등록 후 sudo dnf makecache로 다시 캐시를 생성하고, sudo dnf install -y dkms nvidia-driver 명령어로 설치를 시도하세요. [5] 

[1] [https://velog.io](https://velog.io/@ksm0517/Rocky-Linux-Error-during-download-metadata)
[2] [https://velog.io](https://velog.io/@ksm0517/Rocky-Linux-Error-during-download-metadata)
[3] [https://docs.rockylinux.org](https://docs.rockylinux.org/10/ko/desktop/display/installing_nvidia_gpu_drivers/)
[4] [https://www.reddit.com](https://www.reddit.com/r/RockyLinux/comments/1c9xd1r/new_to_rocky_linux_can_not_install_nvidia/?tl=ko)
[5] [https://thelinuxcluster.com](https://thelinuxcluster.com/2022/04/26/installing-nvidia-drivers-on-rocky-linux-8-5/)









코드를 사용할 때는 주의가 필요합니다.4. 재부팅 및 확인설치가 완료되면 시스템을 재부팅합니다.bashsudo reboot
코드를 사용할 때는 주의가 필요합니다.
```