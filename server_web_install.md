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

sudo systemctl start httpd

# 상태 확인
sudo systemctl status httpd
```

4. 방화벽 적용

```bash
sudo firewall-cmd --permanent --zone=public --add-port=80/tcp
sudo firewall-cmd --reload

sudo firewall-cmd --list-all
```

5. 배포시
   - rpm 설치 시 소유자 변경은 필요 없음.
   - 단, html, css 등 웹 애플리케이션 파일을 배포할 때는 apache 계정이 읽을 수 있도록 권한을 맞춰야함.
     ```bash
     sudo chown -R apache:apache /var/www/html/myapp
     sudo chmod -R 755 /var/www/html/myapp
     ```
---

## 2. OpenJDK 21 설치

1. 파일 다운로드 = `https://jdk.java.net/archive/`

2. 파일 압축해체

```bash
sudo tar -xvf openjdk-21_linux-x64_bin.tar.gz -C /opt/

```

3. 환경변수 설정

```bash
# 1. /etc/profile 또는 .bashrc 열기
sudo nano /etc/profile.d/java.sh

# 2. 하단에 아래 내용 추가
export JAVA_HOME=/opt/jdk-21.0.2
export PATH=$JAVA_HOME/bin:$PATH

# 3. 적용
source /etc/profile.d/java.sh

# 4. 제대로 설치되었는지 확인
java -version
javac -version
```

---

## 3. tomcat 설치

1. tomcat 설치
```bash
sudo cp apache-tomcat-10.1.60.tar.gz /opt/
cd /opt/
sudo tar -zxvf apache-tomcat-10.1.60.tar.gz
sudo mv apache-tomcat-10.1.60 tomcat

```

2. 계정 생성 후 서비스 적용

    1. `tomcat.service` 생성 = `sudo vi /etc/systemd/system/tomcat.service`
    2. 아래 파일 작성 후 저장
        ```bash
          [Unit]
          Description=Apache Tomcat 10.1.60
          After=network.target
          
          [Service]
          Type=forking
          User=tomcat
          Group=tomcat
          Environment="JAVA_HOME=/opt/jdk-21.0.2"
          Environment="CATALINA_HOME=/opt/tomcat"
          Environment="CATALINA_BASE=/opt/tomcat"
          ExecStart=/opt/tomcat/bin/startup.sh
          ExecStop=/opt/tomcat/bin/shutdown.sh
          Restart=on-failure
          
          [Install]
          WantedBy=multi-user.target
        ```
    
    3. 서비스 반영 및 실행
       ```bash
       sudo systemctl daemon-reload
       sudo systemctl enable --now tomcat
       sudo systemctl status tomcat
       sudo systemctl start tomcat
       sudo systemctl stop tomcat
       ```

    4. 방화벽 설정
       ```bash
       sudo firewall-cmd --permanent --zone=public --add-port=8080/tcp
       sudo firewall-cmd --reload
       ```
---

## 4. 계정

1. 계정 확인 = `cat /etc/passwd`
2. 톰캣 계정 생성        
    1. 그룹 생성 = `sudo groupadd tomcat`
    2. 홈 디렉토리 없이 계정 생성: `sudo useradd -r -g tomcat -d /opt/tomcat-10/ -s /usr/sbin/nologin tomcat`
    3. 권한 설정 = `sudo chown -R tomcat:tomcat /opt/tomcat-10/`
    4. 실행권한 부여 = `sudo chmod +x /opt/tomcat-10/bin/*.sh`
     
---

## 5. Apache Web 과 연동

1. 모듈 확인

```bash
# 1. 모듈 설정 파일 확인:
ls /etc/httpd/conf.modules.d/
# → 모듈별 .conf 파일들이 보입니다.

# 실제 로드된 모듈 확인:
httpd -M
# → 현재 Apache가 로드한 모듈 목록 출력(shared: 활성화)

# 2. Proxy 모듈 활성화
# mod_proxy
# mod_proxy_http
# mod_proxy_ajp (AJP 프로토콜 사용 시)

# 3. 확인
httpd -M | grep proxy

```

2. Apache 설정 파일 수정 = `sudo vi /etc/httpd/conf.d/tomcat.conf`

```bash
<VirtualHost *:80>
    ServerName localhost

    ProxyRequests Off
    ProxyPreserveHost On

    <Proxy *>
        Require all granted
    </Proxy>

    ProxyPass /app http://localhost:8080/app
    ProxyPassReverse /app http://localhost:8080/app
</VirtualHost>
# /app → Tomcat에서 서비스하는 웹 애플리케이션 경로
# http://localhost:8080 → Tomcat 기본 포트
```

3. `httpd.conf` 파일 수정
```bash
# 제일 하단에 추가(필요시)
ServerName localhost:80

```

4. 아파치 재시작= `sudo systemctl restart httpd`
