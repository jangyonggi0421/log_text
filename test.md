## 1. 현재 109번 구성 확인
Tomcat 서비스 이름 확인:
```bash
systemctl list-unit-files \
  | grep -i tomcat
```
실행 중인 Tomcat 경로 확인:
```bash
ps -eo pid,args \
  | grep '[o]rg.apache.catalina.startup.Bootstrap'
```
서비스 설정 확인:
```bash
sudo systemctl cat tomcat
```
Apache ProxyPass 확인:
```bash
sudo grep -RniE \
  'ProxyPass|ProxyPassReverse|18080|8080' \
  /etc/httpd/conf \
  /etc/httpd/conf.d
```
DB 경로 확인:
```bash
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "SHOW data_directory;"
```

## 2. 기존 설정 백업
```bash
sudo install -d \
  -o root \
  -g root \
  -m 700 \
  /data/tsinspec/backup/109-before-data-migration
```
Apache 설정 백업:
```bash
sudo cp -a \
  /etc/httpd/conf.d \
  /data/tsinspec/backup/109-before-data-migration/httpd-conf.d
```
현재 Tomcat 서비스 파일이 있다면 백업합니다.
```bash 
sudo cp -a \
  /etc/systemd/system/tomcat.service \
  /data/tsinspec/backup/109-before-data-migration/
```
환경변수도 백업하되 권한을 제한합니다.
```bash
sudo cp -a \
  /etc/sysconfig/tsinspec-tomcat \
  /data/tsinspec/backup/109-before-data-migration/

sudo chmod 600 \
  /data/tsinspec/backup/109-before-data-migration/tsinspec-tomcat
```

## 3. `/data` 운영 디렉터리 생성
```bash
sudo install -d \
  -o root \
  -g tomcat \
  -m 750 \
  /data/tsinspec \
  /data/tsinspec/apps \
  /data/tsinspec/deploy \
  /data/tsinspec/backup \
  /data/tsinspec/tomcat-base \
  /data/tsinspec/tomcat-base/conf \
  /data/tsinspec/tomcat-base/lib \
  /data/tsinspec/conf
```
Tomcat 기록 경로:
```bash
sudo install -d \
  -o tomcat \
  -g tomcat \
  -m 750 \
  /data/tsinspec/tomcat-base/logs \
  /data/tsinspec/tomcat-base/temp \
  /data/tsinspec/tomcat-base/webapps \
  /data/tsinspec/tomcat-base/work \
  /data/tsinspec/data \
  /data/tsinspec/data/upload \
  /data/tsinspec/data/sync
```

## 4. Tomcat 설치파일 배치
기존 Tomcat 설치 경로를 이동하지 말고 `tar.gz` 원본을 `/data`에 복사합니다.
```bash
sudo cp \
  /기존경로/apache-tomcat-10.1.60.tar.gz \
  /data/tsinspec/deploy/
```
압축 확인:
```bash
tar -tzf \
  /data/tsinspec/deploy/apache-tomcat-10.1.60.tar.gz \
  | head
```
압축 해제:
```bash
sudo tar -xzf \
  /data/tsinspec/deploy/apache-tomcat-10.1.60.tar.gz \
  -C /data/tsinspec/apps
```
권한 설정:
```bash
sudo chown -R root:tomcat \
  /data/tsinspec/apps/apache-tomcat-10.1.60

sudo find \
  /data/tsinspec/apps/apache-tomcat-10.1.60 \
  -type d \
  -exec chmod 750 {} \;

sudo find \
  /data/tsinspec/apps/apache-tomcat-10.1.60 \
  -type f \
  -exec chmod 640 {} \;

sudo chmod 750 \
  /data/tsinspec/apps/apache-tomcat-10.1.60/bin/*.sh
```
심볼릭 링크:
```bash
sudo ln -sfn \
  /data/tsinspec/apps/apache-tomcat-10.1.60 \
  /data/tsinspec/apps/tomcat
```
## 5. Tomcat 설정 복사
기존 Tomcat이 CATALINA_HOME과 CATALINA_BASE를 분리하지 않았다면:
```bash
sudo cp -a \
  <기존_TOMCAT_경로>/conf/. \
  /data/tsinspec/tomcat-base/conf/
```
권한 설정:
```bash
sudo chown -R root:tomcat \
  /data/tsinspec/tomcat-base/conf

sudo find \
  /data/tsinspec/tomcat-base/conf \
  -type d \
  -exec chmod 750 {} \;

sudo find \
  /data/tsinspec/tomcat-base/conf \
  -type f \
  -exec chmod 640 {} \;
```
server.xml에서 포트를 확인합니다.(9910)
```bash
sudo vi \
  /data/tsinspec/tomcat-base/conf/server.xml
```
권장 설정:
```xml
<Connector
    address="127.0.0.1"
    port="9910"
    protocol="HTTP/1.1"
    connectionTimeout="20000"
    URIEncoding="UTF-8"
    maxParameterCount="1000"
    redirectPort="8443" />
```

## 6. WAR와 파일 데이터 복사
기존 WAR를 복사합니다.
```bash
sudo cp \
  <기존_WAR_경로>/tsinspec.war \
  /data/tsinspec/tomcat-base/webapps/tsinspec.war
```
권한 설정:
```bash
sudo chown tomcat:tomcat \
  /data/tsinspec/tomcat-base/webapps/tsinspec.war

sudo chmod 640 \
  /data/tsinspec/tomcat-base/webapps/tsinspec.war
```
기존 업로드 파일이 있다면 /data로 복사합니다.
```bash
sudo cp -a \
  <기존_업로드_경로>/. \
  /data/tsinspec/data/upload/

sudo chown -R tomcat:tomcat \
  /data/tsinspec/data
```

## 7. 환경변수 설정
/etc/sysconfig/tsinspec-tomcat:
```ini
JAVA_OPTS="-Dfile.encoding=UTF-8 -Duser.timezone=Asia/Seoul -Djava.awt.headless=true"

CATALINA_OPTS="-Xms1024m -Xmx2048m -XX:+ExitOnOutOfMemoryError -Dspring.profiles.active=linux"

TSINSPEC_DB_URL="jdbc:postgresql://127.0.0.1:32419/ts_insp_db?currentSchema=inspection&ApplicationName=tsinspec&connectTimeout=5"
TSINSPEC_DB_USERNAME="tsins_user"
TSINSPEC_DB_PASSWORD="<현재 109번 DB 비밀번호>"

TSINSPEC_PHONE_ENCRYPTION_KEY="1KS+RGWNATVWTmlEnJj5QQVP1nrBsRaddNxXE6zkVM4="

TSINSPEC_DATA_DIR="/data/tsinspec/data"
TSINSPEC_CONF_DIR="/data/tsinspec/conf"
TSINSPEC_LOG_DIR="/data/tsinspec/tomcat-base/logs"
TSINSPEC_FILE_STORE_DIR="/data/tsinspec/data/upload"
TSINSPEC_SYNC_DIR="/data/tsinspec/data/sync"

TSINSPEC_SSO_ENABLED="false"
```
기존 DB와 암호화 키 값은 변경하지 않습니다.
```bash
sudo chown root:root \
  /etc/sysconfig/tsinspec-tomcat

sudo chmod 600 \
  /etc/sysconfig/tsinspec-tomcat
```

## 8. 신규 Tomcat 서비스 생성
/etc/systemd/system/tsinspec-tomcat.service:
```ini
[Unit]
Description=TS Inspection Tomcat 10.1.60
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk"
Environment="CATALINA_HOME=/data/tsinspec/apps/tomcat"
Environment="CATALINA_BASE=/data/tsinspec/tomcat-base"
Environment="CATALINA_TMPDIR=/data/tsinspec/tomcat-base/temp"

EnvironmentFile=-/etc/sysconfig/tsinspec-tomcat

WorkingDirectory=/data/tsinspec/tomcat-base
ExecStart=/data/tsinspec/apps/tomcat/bin/catalina.sh run

Restart=on-failure
RestartSec=10
TimeoutStartSec=120
TimeoutStopSec=60
KillSignal=SIGTERM
SuccessExitStatus=143
UMask=0027
LimitNOFILE=65535
NoNewPrivileges=true
PrivateTmp=true

StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```
검사:
```bash
sudo systemd-analyze verify \
  /etc/systemd/system/tsinspec-tomcat.service
sudo systemctl daemon-reload
sudo systemctl enable tsinspec-tomcat
```

## 9. 설정 검사
```bash
sudo -u tomcat env \
  JAVA_HOME=/usr/lib/jvm/java-21-openjdk \
  CATALINA_HOME=/data/tsinspec/apps/tomcat \
  CATALINA_BASE=/data/tsinspec/tomcat-base \
  /data/tsinspec/apps/tomcat/bin/catalina.sh configtest
```

## 10. 전환 테스트
점검 시간을 확보한 뒤 Apache를 잠시 중지합니다.
```bash
sudo systemctl stop httpd
```
기존 Tomcat을 중지합니다.
```bash
sudo systemctl stop <기존_TOMCAT_서비스명>
```
신규 Tomcat을 시작합니다.
```bash
sudo systemctl start tsinspec-tomcat
```
확인:
```bash
sudo systemctl status \
  tsinspec-tomcat \
  --no-pager

sudo journalctl \
  -u tsinspec-tomcat \
  -n 300 \
  --no-pager

curl -I \
  http://127.0.0.1:9910/tsinspec/auth/login.do
```

## 11. Apache 설정

설정파일: `/etc/httpd/conf.d/tsinspec.conf`

```conf
<VirtualHost *:8099>
    ServerName localhost
    ServerAlias localhost

    ProxyRequests Off
    ProxyPreserveHost On

    RewriteEngine On
    RewriteRule ^/?$ /tsinspec/ [R=302,L]

    ProxyPass        /tsinspec/ http://127.0.0.1:9910/tsinspec/ connectiontimeout=5 timeout=120
    ProxyPassReverse    /tsinspec/      http://127.0.0.1:9910/tsinspec/

    <LocationMatch "^/tsinspec(?:/|$)">
        <RequireAny>
            Require local
            Require ip 192.168.196.105/32
            Require ip 192.168.196.106/32
            Require ip 192.168.196.150/32
        </RequireAny>
    </LocationMatch>
</VirtualHost>

# 2안
 <VirtualHost *:8099>
        ServerName <운영_IP_또는_호스트명>
    
        ProxyRequests Off
        ProxyPreserveHost On
        ProxyAddHeaders On
    
        RequestHeader set X-Forwarded-Proto "http"
        RequestHeader set X-Forwarded-Port "8099"
    
        RedirectMatch 302 "^/$" "/tsinspec/auth/login.do"
        RedirectMatch 302 "^/tsinspec$" "/tsinspec/auth/login.do"
    
        ProxyPass        "/tsinspec/" \
                         "http://127.0.0.1:9910/tsinspec/" \
                         connectiontimeout=5 timeout=120
    
        ProxyPassReverse "/tsinspec/" \
                         "http://127.0.0.1:9910/tsinspec/"
    
        <Location "/tsinspec/">
            Require ip 127.0.0.1
            Require ip <허용할_사용자_IP>
            Require ip <허용할_네트워크/CIDR>
        </Location>
    
        ErrorLog  /var/log/httpd/tsinspec_error.log
        CustomLog /var/log/httpd/tsinspec_access.log combined
    </VirtualHost>
```

설정 검사: `sudo httpd -t`

정상이면: `sudo systemctl start httpd`

최종 접속: `curl -i http://127.0.0.1/tsinspec/auth/login.do`

## 12. 롤백 방법
신규 Tomcat에서 문제가 발생하면:
```bash
sudo systemctl stop httpd
sudo systemctl stop tsinspec-tomcat
```

Apache ProxyPass를 기존 Tomcat 포트로 되돌립니다.

```bash
sudo httpd -t
sudo systemctl start <기존_TOMCAT_서비스명>
sudo systemctl start httpd
```

DB를 변경하지 않았기 때문에 Tomcat과 Apache 설정만 되돌리면 됩니다.

- 테스트 완료 후에도 유지할 항목
  - 정상 확인 후에도 바로 삭제하지 않습니다.
    - 기존 Tomcat 설치 디렉터리
    - 기존 Tomcat 서비스 파일
    - 기존 Apache 설정 백업
    - 기존 WAR
    - 기존 업로드 파일

109번에서 며칠간 정상 운영을 확인한 뒤 신규 VM에도 동일한 /data/tsinspec 구조를 적용하면 됩니다. PostgreSQL과 DNF 설치 패키지는 이번 테스트 범위에서 그대로 유지하면 됩니다.

## 13. DB 경로 확인 및 백업, 복구
```text
# /data/postgresql/18/data       PostgreSQL 운영 데이터·설정
# /data/postgresql/18/backup     논리 백업
# /data/postgresql/18/archive    WAL 아카이브
# /data/tsinspec                 Tomcat·WAR·업로드 데이터
```

PostgreSQL 프로그램과 서비스는 DNF 설치 경로를 유지합니다.
```text
/usr/pgsql-18
postgresql-18.service
```

1. 현재 DB 경로 확인
```bash
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "SHOW data_directory;"
```

이미 다음 결과라면 데이터 이동은 필요 없습니다.
`/data/postgresql/18/data`

다음과 같이 나오면 이전 작업을 진행합니다.
`/var/lib/pgsql/18/data`

현재 포트와 데이터 체크섬도 확인합니다.
```bash
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "SHOW port;"

sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "SHOW data_checksums;"
```

2. 외부 테이블스페이스와 WAL 경로 확인
기본 경로 밖의 테이블스페이스가 있는지 확인합니다.
```bash
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "
SELECT
    spcname,
    pg_tablespace_location(oid) AS location
FROM pg_tablespace
ORDER BY spcname;
"
```

pg_default, pg_global만 있고 location이 비어 있으면 일반적인 구성입니다.
WAL이 외부 경로로 연결됐는지도 확인합니다.

```bash
sudo ls -ld \
  /var/lib/pgsql/18/data/pg_wal

sudo readlink -f \
  /var/lib/pgsql/18/data/pg_wal
```

다음처럼 데이터 디렉터리 내부 경로이면 그대로 복사됩니다.
`/var/lib/pgsql/18/data/pg_wal`

별도 경로가 나온다면 해당 WAL 경로도 별도로 /data로 이전해야 하므로 바로 복사하지 말고 경로를 먼저 정리해야 합니다.

3. 이전 전 논리 백업
Tomcat을 중지하여 DB 쓰기를 차단합니다.
```bash
sudo systemctl stop tsinspec-tomcat
```

기존 서비스 이름을 아직 사용한다면 해당 서비스를 중지합니다.
```bash
sudo systemctl stop tomcat
```

DB는 실행 중인 상태에서 백업 디렉터리를 만듭니다.
```bash
sudo install -d \
  -o postgres \
  -g postgres \
  -m 700 \
  /data/postgresql/18/backup
```

애플리케이션 DB 백업:
```bash
sudo -u postgres /usr/pgsql-18/bin/pg_dump \
  -p 32419 \
  -d ts_insp_db \
  --format=custom \
  --compress=6 \
  --verbose \
  --file=/data/postgresql/18/backup/ts_insp_db_before_pgdata_move_20261015.dump
```

PostgreSQL 역할 백업:
```bash
sudo -u postgres /usr/pgsql-18/bin/pg_dumpall \
  -p 32419 \
  --globals-only \
  --file=/data/postgresql/18/backup/globals_before_pgdata_move_20261015.sql
```

역할 백업에는 PostgreSQL 비밀번호 검증값이 들어갈 수 있으므로 권한을 제한합니다.
```bash
sudo chmod 600 \
  /data/postgresql/18/backup/ts_insp_db_before_pgdata_move_20261015.dump \
  /data/postgresql/18/backup/globals_before_pgdata_move_20261015.sql
```

체크섬:
```bash
cd /data/postgresql/18/backup

sudo -u postgres sha256sum \
  ts_insp_db_before_pgdata_move_20261015.dump \
  globals_before_pgdata_move_20261015.sql \
  | sudo tee pgdata_move_20261015.sha256 >/dev/null
```

4. 이전 전 데이터 건수 기록
```bash
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d ts_insp_db
SELECT count(*) AS user_count
FROM inspection.tbl_user_account;

SELECT count(*) AS asset_count
FROM inspection.tbl_asset;

SELECT count(*) AS audit_count
FROM inspection.tbl_audit_log;

SELECT count(*) AS login_attempt_count
FROM inspection.tbl_login_attempt;

\q
```
결과를 별도로 기록합니다.

5. 신규 데이터 경로 생성
먼저 기존 상태를 확인합니다.
```bash
sudo ls -la \
  /data/postgresql/18/data
```

디렉터리가 없으면 생성합니다.
```bash
sudo install -d \
  -o postgres \
  -g postgres \
  -m 700 \
  /data/postgresql/18/data
```

이미 다른 PostgreSQL 클러스터가 초기화되어 있다면 기존 데이터와 섞으면 안 됩니다. 별도로 보관합니다.
```bash
sudo mv \
  /data/postgresql/18/data \
  /data/postgresql/18/data-before-migration-20261015
```

그다음 빈 디렉터리를 다시 만듭니다.
```bash
sudo install -d \
  -o postgres \
  -g postgres \
  -m 700 \
  /data/postgresql/18/data
```

6. SELinux 경로 등록
```bash
sudo semanage fcontext -a \
  -t postgresql_db_t \
  '/data/postgresql/18(/.*)?'
```

이미 등록된 규칙이라는 오류가 나오면:
```bash
sudo semanage fcontext -m \
  -t postgresql_db_t \
  '/data/postgresql/18(/.*)?'
```

적용:
```bash
sudo restorecon -RFv \
  /data/postgresql/18
```

확인:
```bash
ls -Zd \
  /data/postgresql/18 \
  /data/postgresql/18/data
```
semanage가 없다면 먼저 설치합니다.
```bash
sudo dnf install policycoreutils-python-utils
```

7. PostgreSQL 완전 중지
```bash
sudo systemctl stop postgresql-18
```

상태 확인:
```bash
sudo systemctl status \
  postgresql-18 \
  --no-pager
```

프로세스가 남아 있지 않은지 확인합니다.
```bash
ps -ef | grep '[p]ostgres'
```
결과가 없어야 합니다.

8. 기존 데이터 디렉터리 복사

rsync가 있는지 확인합니다.
```bash
command -v rsync
```

없으면 설치합니다.
```bash
sudo dnf install rsync
```

기존 데이터 디렉터리가 /var/lib/pgsql/18/data인 경우:
```bash
sudo rsync \
  -aHAX \
  --numeric-ids \
  /var/lib/pgsql/18/data/ \
  /data/postgresql/18/data/
```

소유권과 권한을 다시 정리합니다.
```bash
sudo chown -R postgres:postgres \
  /data/postgresql/18/data

sudo chmod 700 \
  /data/postgresql/18/data
```

디스크에 기록을 반영합니다.
```bash
sync
```

두 디렉터리의 크기를 비교합니다.
```bash
sudo du -sh \
  /var/lib/pgsql/18/data \
  /data/postgresql/18/data
```

9. 새 데이터 디렉터리 검사
PostgreSQL 버전 정보 확인:
```bash
sudo -u postgres /usr/pgsql-18/bin/pg_controldata \
  /data/postgresql/18/data \
  | grep -E \
  'Database cluster state|Database block size|Data page checksum version'
```

정상 종료 상태는 다음과 같습니다.
```text
Database cluster state: shut down
```

`in production` 또는 `in crash recovery`라면 서비스를 시작하지 말고 복사 과정을 다시 확인해야 합니다.

11. systemd의 PGDATA 변경
직접 서비스 원본 파일을 수정하지 않고 drop-in을 생성합니다.
```bash
sudo systemctl edit --full postgresql-18
```

다음 내용을 입력합니다.
```bash
[Service]
Environment=PGDATA=/data/postgresql/18/data
```

적용:
```bash
sudo systemctl daemon-reload
```

확인:
```bash
sudo systemctl show \
  postgresql-18 \
  --property=Environment
```

다음 값이 포함되어야 합니다.
```text
PGDATA=/data/postgresql/18/data
```

12. PostgreSQL 시작
```bash
sudo systemctl start postgresql-18
```

상태 확인:
```bash
sudo systemctl status \
  postgresql-18 \
  --no-pager
```

문제가 발생하면:
```bash
sudo journalctl \
  -u postgresql-18 \
  -n 300 \
  --no-pager
```

13. 변경 결과 확인
데이터 경로:
```bash
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "SHOW data_directory;"
```

정상 결과:
```bash
/data/postgresql/18/data
```

포트:
```bash
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "SHOW port;"
```

DB 및 역할:
```bash
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "\l+"

sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "\du+"
```

테이블 확인:
```bash
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d ts_insp_db \
  -c "\dt inspection.*"
```
앞서 기록한 데이터 건수도 다시 비교합니다.

13. 설정 내 기존 경로 확인
새 데이터 디렉터리 안에 기존 /var/lib/pgsql 경로가 남아 있는지 확인합니다.
```bash
sudo grep -Rni \
  '/var/lib/pgsql' \
  /data/postgresql/18/data \
  --exclude='postmaster.pid'
```

다음 설정에서 기존 경로가 발견되면 /data 기준으로 수정합니다.
- archive_command
- restore_command
- log_directory
- 외부 SSL 인증서 경로
- 사용자 정의 include 파일
- 테이블스페이스 경로

WAL 아카이브를 사용할 경우:
```text
archive_mode = on
archive_command = 'test ! -f /data/postgresql/18/archive/%f && cp %p /data/postgresql/18/archive/%f'
```

아카이브 디렉터리:
```bash
sudo install -d \
  -o postgres \
  -g postgres \
  -m 700 \
  /data/postgresql/18/archive

sudo restorecon -RFv \
  /data/postgresql/18/archive
```
WAL 아카이브를 사용하지 않는다면 archive_mode=off 상태로 유지합니다.

14. Tomcat 다시 시작
DB 확인이 끝나면 /data 기반 Tomcat을 시작합니다.
```bash
sudo systemctl start tsinspec-tomcat
```

확인:
```bash
sudo journalctl \
  -u tsinspec-tomcat \
  -n 300 \
  --no-pager
```

직접 접속:
```bash
curl -I http://127.0.0.1:18080/tsinspec/auth/login.do
```

15. 롤백 방법
신규 PGDATA에서 문제가 발생하면 원본 데이터 디렉터리를 삭제하지 않았으므로 되돌릴 수 있습니다.

```bash
# PostgreSQL 중지:
sudo systemctl stop postgresql-18

# 현재 override 파일을 비활성화합니다.
sudo mv \
  /etc/systemd/system/postgresql-18.service.d/override.conf \
  /etc/systemd/system/postgresql-18.service.d/override.conf.disabled

# 적용:
sudo systemctl daemon-reload
sudo systemctl start postgresql-18

# 확인:
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "SHOW data_directory;"
```

기존 경로가 나오면 롤백된 것입니다.

삭제하지 말아야 할 경로
```text
/var/lib/pgsql/18/data
/data/postgresql/18/data-before-migration-20261015
/data/postgresql/18/backup
```

109번에서 /data 기반 Tomcat·PostgreSQL이 정상 동작하고 로그인, 자산, 점검, 보고서 기능까지 검증한 후에 기존 PostgreSQL 경로 정리를 검토하면 됩니다.

16. `pg_hba.conf` 설정
```conf
# pg_hba.conf
host    ts_insp_db      tsins_user      192.168.196.0/24        scram-sha-256
local   all             all                                     scram-sha-256
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256
```

## 14. 오류 시
> tsinspec-tomcat.service: Failed at step EXEC spawning /data/tsinspec/apps/tomcat/bin/catalina.sh: Permission denied 왜 이러지?

이 오류는 Java나 Tomcat 설정을 읽기 전, systemd가 catalina.sh 자체를 실행하지 못한 상태입니다. 주요 원인은 다음 셋 중 하나입니다.

1. catalina.sh 실행 권한 누락
2. /data가 noexec로 마운트됨
3. SELinux가 /data의 실행을 차단함

순서대로 확인하면 됩니다.

1. 실행 권한과 상위 경로 확인
```bash
readlink -f \
  /data/tsinspec/apps/tomcat

# 예상 결과:
/data/tsinspec/apps/apache-tomcat-10.1.60

# 스크립트 권한 확인:
ls -lZ \
  /data/tsinspec/apps/tomcat/bin/catalina.sh

# 전체 상위 경로 권한 확인:
namei -l \
  /data/tsinspec/apps/tomcat/bin/catalina.sh

# tomcat 계정이 실행할 수 있는지 확인합니다.
sudo -u tomcat test \
  -x /data/tsinspec/apps/tomcat/bin/catalina.sh

echo $?
# - 0: 권한상 실행 가능
# - 1: 실행 권한 또는 상위 디렉터리 접근 권한 문제
```

2. Tomcat 권한 재설정
```bash
sudo chown -R root:tomcat \
  /data/tsinspec/apps/apache-tomcat-10.1.60

# 디렉터리 권한:
sudo find \
  /data/tsinspec/apps/apache-tomcat-10.1.60 \
  -type d \
  -exec chmod 750 {} \;

# 일반 파일 권한:
sudo find \
  /data/tsinspec/apps/apache-tomcat-10.1.60 \
  -type f \
  -exec chmod 640 {} \;

# 스크립트에 실행 권한을 다시 부여합니다.
sudo chmod 750 \
  /data/tsinspec/apps/apache-tomcat-10.1.60/bin/*.sh

# 상위 디렉터리도 접근 가능하게 설정합니다.
sudo chown root:tomcat \
  /data/tsinspec \
  /data/tsinspec/apps

sudo chmod 750 \
  /data/tsinspec \
  /data/tsinspec/apps

# /data 자체 권한도 확인합니다.
ls -ld /data

# tomcat 계정에 /data 통과 권한이 없으면 ACL로 최소 권한만 부여할 수 있습니다.
sudo setfacl \
  -m u:tomcat:--x \
  /data

# 다시 확인합니다.
sudo -u tomcat test \
  -x /data/tsinspec/apps/tomcat/bin/catalina.sh

echo $?
```

3. SELinux 확인
```bash
getenforce

# Enforcing이면 최근 차단 기록을 확인합니다.
sudo ausearch \
  -m AVC \
  -ts recent \
  | tail -50

# 현재 컨텍스트 확인:
ls -ldZ \
  /data/tsinspec/apps \
  /data/tsinspec/apps/apache-tomcat-10.1.60

ls -lZ \
  /data/tsinspec/apps/tomcat/bin/catalina.sh

# Tomcat 프로그램 경로를 실행 가능한 일반 프로그램 타입으로 영구 등록합니다.
sudo semanage fcontext -a \
  -t usr_t \
  '/data/tsinspec/apps(/.*)?'

# 규칙이 이미 존재한다면:
sudo semanage fcontext -m \
  -t usr_t \
  '/data/tsinspec/apps(/.*)?'

# 적용:
sudo restorecon -RFv \
  /data/tsinspec/apps

# 확인:
ls -lZ \
  /data/tsinspec/apps/tomcat/bin/catalina.sh

# 컨텍스트에 usr_t가 포함되어야 합니다.
```

5. 스크립트 형식 확인
```bash
file \
  /data/tsinspec/apps/tomcat/bin/catalina.sh

# 첫 줄 확인:
head -1 \
  /data/tsinspec/apps/tomcat/bin/catalina.sh

# 정상:
#!/bin/sh

# Windows CRLF가 포함되면 다음 명령으로 변환합니다.
sudo sed -i 's/\r$//' \
  /data/tsinspec/apps/apache-tomcat-10.1.60/bin/*.sh

# 변환 후 실행 권한을 다시 적용합니다.
sudo chmod 750 \
  /data/tsinspec/apps/apache-tomcat-10.1.60/bin/*.sh
```

7. systemd 없이 직접 설정 검사
```bash
sudo -u tomcat env \
  JAVA_HOME=/usr/lib/jvm/java-21-openjdk \
  CATALINA_HOME=/data/tsinspec/apps/tomcat \
  CATALINA_BASE=/data/tsinspec/tomcat-base \
  /data/tsinspec/apps/tomcat/bin/catalina.sh configtest
```
이 명령이 성공해야 systemd에서도 실행할 수 있습니다.

8. 서비스 재시작
```bash
sudo systemctl daemon-reload
sudo systemctl restart tsinspec-tomcat

# 확인:
sudo systemctl status \
  tsinspec-tomcat \
  --no-pager

sudo journalctl \
  -u tsinspec-tomcat \
  -n 200 \
  --no-pager

# 가장 먼저 다음 두 결과를 확인하면 원인을 빠르게 구분할 수 있습니다.
sudo -u tomcat test \
  -x /data/tsinspec/apps/tomcat/bin/catalina.sh
echo $?

findmnt -no OPTIONS /data
```
실행 권한 결과가 0인데도 systemd에서 계속 Permission denied가 발생하면 /data의 noexec 또는 SELinux가 원인일 가능성이 가장 높습니다.

---

## 톰캣, 아파치 등 설정 및 수정

1. 실제 CATALINA_BASE 확인
```bash
sudo systemctl cat tsinspec-tomcat
```

다음 설정을 확인합니다.
`Environment="CATALINA_BASE=/data/tsinspec/tomcat-base"`

그러면 수정해야 하는 파일은 다음입니다.
`/data/tsinspec/tomcat-base/conf/server.xml`

Tomcat 원본 경로의 다음 파일을 수정해도 적용되지 않을 수 있습니다.
`/data/tsinspec/apps/tomcat/conf/server.xml`

2. Connector 설정 확인
```bash
sudo grep -nA8 -B2 \
  'port="9910"' \
  /data/tsinspec/tomcat-base/conf/server.xml

# 권장 설정:
<Connector
    address="127.0.0.1"
    port="9910"
    protocol="HTTP/1.1"
    connectionTimeout="20000"
    URIEncoding="UTF-8"
    maxParameterCount="1000"
    redirectPort="8443" />
```
Tomcat이 별도 VM의 Apache 요청을 받아야 하는 구조라면 127.0.0.1이 아닌 Tomcat 서버 내부 IP로 설정해야 합니다. Apache와 Tomcat이 같은 VM이면 127.0.0.1이 맞습니다.

3. 포트 충돌 확인
Tomcat을 중지한 상태에서 확인합니다.
```bash
sudo systemctl stop tsinspec-tomcat

sudo ss -lntp \
  | grep ':9910'
```
결과가 없어야 합니다. 다른 프로세스가 이미 9910 포트를 사용하고 있다면 Tomcat이 시작되지 않습니다.

4. Tomcat 설정 검사
```bash
sudo -u tomcat env \
  JAVA_HOME=<실제_JDK_디렉터리> \
  CATALINA_HOME=/data/tsinspec/apps/tomcat \
  CATALINA_BASE=/data/tsinspec/tomcat-base \
  /data/tsinspec/apps/tomcat/bin/catalina.sh configtest

# 예:
sudo -u tomcat env \
  JAVA_HOME=/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64 \
  CATALINA_HOME=/data/tsinspec/apps/tomcat \
  CATALINA_BASE=/data/tsinspec/tomcat-base \
  /data/tsinspec/apps/tomcat/bin/catalina.sh configtest
```
오류가 발생하면 server.xml 문법부터 수정해야 합니다.

5. WAR 위치와 권한 확인
```bash
sudo ls -lh \
  /data/tsinspec/tomcat-base/webapps/tsinspec.war

# 정상 권한 예:
-rw-r----- tomcat tomcat tsinspec.war

# 수정:
sudo chown tomcat:tomcat \
  /data/tsinspec/tomcat-base/webapps/tsinspec.war

sudo chmod 640 \
  /data/tsinspec/tomcat-base/webapps/tsinspec.war

# WAR 내부 확인:
jar tf \
  /data/tsinspec/tomcat-base/webapps/tsinspec.war \
  | grep 'WEB-INF/web.xml'
```

6. 기존 압축 해제 디렉터리 정리
기존 tsinspec 디렉터리가 남아 있으면 예전 파일과 신규 WAR가 섞일 수 있습니다. 삭제하지 말고 백업 위치로 이동합니다.
```bash
TSINSPEC_DEPLOY_TIME=$(date +%Y%m%d_%H%M%S)

sudo mv \
  /data/tsinspec/tomcat-base/webapps/tsinspec \
  /data/tsinspec/backup/tsinspec-expanded-${TSINSPEC_DEPLOY_TIME}
```
tsinspec 디렉터리가 존재할 때만 실행합니다.

7. Tomcat 시작
```bash
sudo systemctl daemon-reload

sudo systemctl reset-failed tsinspec-tomcat

sudo systemctl start tsinspec-tomcat

# 상태 확인:
sudo systemctl status \
  tsinspec-tomcat \
  --no-pager

# 배포 로그:
sudo journalctl \
  -u tsinspec-tomcat \
  -n 500 \
  --no-pager
```
다음과 같은 메시지를 확인합니다.
```text
Deploying web application archive
Deployment of web application archive
Server startup
```

8. 9910 포트 확인
```bash
sudo ss -lntp \
  | grep ':9910'

# 정상 예:
127.0.0.1:9910

# 기존 포트도 확인합니다.
sudo ss -lntp \
  | grep -E ':8080|:18080|:9910'
```
Tomcat이 여전히 기존 포트에서 실행된다면 수정한 server.xml이 실제 CATALINA_BASE 설정이 아닐 가능성이 큽니다.

9. Apache를 거치지 않고 직접 확인
```bash
curl -I \
  http://127.0.0.1:9910/tsinspec/auth/login.do

# 또는 상태 코드만 확인합니다.
curl -sS \
  -o /dev/null \
  -w 'HTTP %{http_code}\n' \
  http://127.0.0.1:9910/tsinspec/auth/login.do

# 결과별 의미:
# 결과	의미
# 200, 302	Tomcat과 애플리케이션 정상
# 연결 거부	Tomcat 미기동 또는 Connector 오류
# 404	WAR 미배포 또는 컨텍스트 경로 오류
# 500	애플리케이션 초기화·DB·Spring 오류
# 503	보통 Apache 프록시 또는 애플리케이션 서비스 오류
```

10. Apache 연동시 참고
Apache 외부 포트를 8099, Tomcat 내부 포트를 9910으로 사용한다면 포트 역할을 다음처럼 분리해야 합니다.
사용자 → Apache :8099 → Tomcat 127.0.0.1:9910
Apache의 ProxyPass 대상에 8099를 넣으면 안 되고, Tomcat 포트인 9910을 넣어야 합니다.

    1. 포트 사용 여부 확인
    ```bash
    sudo ss -lntp \
      | grep -E ':8099|:9910'
  
    # 정상 구성은 다음 형태입니다.
    # 127.0.0.1:9910    java/Tomcat
    # 0.0.0.0:8099      httpd
    ```
    Apache를 시작하기 전 다른 프로세스가 8099를 사용하고 있다면 해당 프로세스와 충돌합니다.

    2. Tomcat 직접 확인
    Apache보다 먼저 Tomcat이 정상이어야 합니다.
    ```bash
    curl -I \
      http://127.0.0.1:9910/tsinspec/auth/login.do
    ```
    200 또는 302가 반환되어야 합니다. 연결 거부나 500이면 Apache 설정 전에 Tomcat 문제부터 해결해야 합니다.

    3. Apache Listen 포트 설정
    /etc/httpd/conf/httpd.conf를 확인합니다.
    ```bash
    sudo grep -Rni \
      '^[[:space:]]*Listen' \
      /etc/httpd/conf \
      /etc/httpd/conf.d
    
    # 8099 포트를 추가합니다.
    # Listen 8099

    # 기존 Listen 80을 유지하면 Apache는 80과 8099를 모두 사용합니다.
    # Listen 80
    # Listen 8099

    # 8099만 사용하려면 기존 Listen 80을 주석 처리할 수 있습니다.
    # Listen 80
    # Listen 8099
    ```
    동일한 Listen 8099가 여러 설정 파일에 중복으로 들어가면 안 됩니다.

    4. SELinux에 Apache 포트 등록
    8099가 http_port_t에 등록되어 있는지 확인합니다.
    ```bash
    sudo semanage port -l \
      | grep http_port_t
    
    # 8099가 없으면 등록합니다.
    sudo semanage port -a \
      -t http_port_t \
      -p tcp \
      8099
    ```

    이미 다른 타입으로 등록되었다는 오류가 나오면 현재 등록을 확인합니다.
    ```bash
    sudo semanage port -l \
      | grep -w 8099
    ```

    다른 서비스가 사용하지 않는 포트임이 확인된 경우에만 변경합니다.  
    ```bash
    sudo semanage port -m \
      -t http_port_t \
      -p tcp \
      8099
    ```

    Apache가 Tomcat으로 연결할 수 있도록 설정합니다.
    ```bash
    sudo setsebool -P \
      httpd_can_network_connect \
      1
    ```
    
    5. 방화벽 허용
    8099 포트로 외부 접속해야 하므로 방화벽을 엽니다.
    ```bash
    sudo firewall-cmd --permanent \
      --add-port=8099/tcp
    
    sudo firewall-cmd --reload

    # 확인:
    sudo firewall-cmd \
      --query-port=8099/tcp

    # 예상 결과:
    yes

    # 허용 IP만 접근시키려면 전체 포트 허용 대신 rich rule을 사용하는 것이 더 안전합니다.
    sudo firewall-cmd --permanent \
      --remove-port=8099/tcp

    sudo firewall-cmd --permanent \
      --add-rich-rule='rule family="ipv4" source address="<허용대역/CIDR>" port protocol="tcp" port="8099" accept'

    sudo firewall-cmd --reload
    ```
    
    6. Apache 설정 검사
    ```bash
    sudo httpd -t

    # 정상 결과:
    Syntax OK

    # VirtualHost 인식 확인:
    sudo httpd -S

    # 모듈 확인:
    sudo httpd -M \
      | grep -E 'proxy_module|proxy_http_module|headers_module'
    ```
    
    7. Apache 재시작
    ```bash
    sudo systemctl restart httpd
    sudo systemctl status \
      httpd \
      --no-pager
    
    # 포트 확인:
    sudo ss -lntp \
      | grep ':8099'
    ```
    
    8. 단계별 접속 확인
    ```bash
    # Apache 로컬 접속:
    curl -I \
      http://127.0.0.1:8099/tsinspec/auth/login.do

    # 루트 리다이렉트:
    curl -I \
      http://127.0.0.1:8099/

    # 외부 PC에서는 다음 주소를 사용합니다.
    # http://<서버IP>:8099/tsinspec/auth/login.do

    # 오류별 원인
    # 결과	주요 원인
    # Apache 시작 실패	Listen 중복, SELinux 포트 미등록, 포트 충돌
    # 연결 거부	Apache가 8099에서 대기하지 않음 또는 방화벽
    # 403	<Location>의 Require ip에 사용자 IP가 없음
    # 404	ProxyPass 경로 또는 WAR 컨텍스트 오류
    # 503	Tomcat 9910 미기동 또는 Apache의 Tomcat 연결 차단
    # 200/302	정상

    # Apache 시작 실패 로그:
    sudo journalctl \
      -u httpd \
      -n 200 \
      --no-pager

    # 프록시 오류:
    sudo tail -100 \
      /var/log/httpd/tsinspec_error.log
    ```
    
    가장 먼저 다음 두 명령을 비교하면 됩니다.
    `curl -I http://127.0.0.1:9910/tsinspec/auth/login.do`
    `curl -I http://127.0.0.1:8099/tsinspec/auth/login.do`
    첫 번째는 정상이고 두 번째만 실패하면 Apache의 Listen 8099, SELinux, 방화벽 또는 ProxyPass 설정 문제입니다.

---

## 오류 수정
0.0.0.0:8099 httpd가 없으면 Apache가 8099 포트에서 대기하지 않는 상태입니다. <VirtualHost *:8099>만으로는 부족하고 반드시 Listen 8099가 있어야 합니다.

1. Apache 실행 상태 확인
```bash
sudo systemctl status httpd --no-pager

# 중지 상태라면 로그를 확인합니다.
sudo journalctl -u httpd -n 200 --no-pager
```

2. Listen 설정 확인
```bash
sudo grep -RniE \
  '^[[:space:]]*Listen[[:space:]]+' \
  /etc/httpd/conf \
  /etc/httpd/conf.d
```

다음 설정이 한 번만 있어야 합니다.
`Listen 8099`

`/etc/httpd/conf/httpd.conf` 또는 `/etc/httpd/conf.d/tsinspec.conf`에 추가합니다.
```bash
sudo vi /etc/httpd/conf.d/tsinspec.conf

# 예:
# Listen 8099

# <VirtualHost *:8099>
#    ServerName <서버IP>

#    ProxyRequests Off
#    ProxyPreserveHost On

#    ProxyPass        "/tsinspec/" "http://127.0.0.1:9910/tsinspec/"
#    ProxyPassReverse "/tsinspec/" "http://127.0.0.1:9910/tsinspec/"
# </VirtualHost>
```
설정 파일 이름이 tsinspec.conf.disabled라면 Apache가 읽지 않습니다. 반드시 .conf여야 합니다.

3. 설정 문법 검사
```bash
sudo httpd -t

# 정상 결과:
Syntax OK

# VirtualHost 인식 확인:
sudo httpd -S
```
출력에 *:8099 VirtualHost가 표시되어야 합니다.

4. 포트 충돌 확인
```bash
sudo ss -lntp | grep ':8099'

# 다른 프로세스가 사용 중이면 Apache가 포트를 열 수 없습니다.
sudo lsof -nP -iTCP:8099 -sTCP:LISTEN
```

5. SELinux 포트 등록
```bash
sudo semanage port -l | grep -w 8099

# 결과가 없으면:
sudo semanage port -a \
  -t http_port_t \
  -p tcp \
  8099

# 이미 다른 SELinux 타입에 등록되어 있다면 해당 서비스 사용 여부를 확인한 후 변경합니다.
sudo semanage port -m \
  -t http_port_t \
  -p tcp \
  8099
```

6. Apache 재시작
```bash
sudo systemctl restart httpd

sudo systemctl status httpd --no-pager

# 다시 확인합니다.
sudo ss -lntp | grep ':8099'

# 정상적인 결과는 IPv4 또는 IPv6 형태 중 하나로 나올 수 있습니다.
# `0.0.0.0:8099`

# 또는:
# [::]:8099
# [::]:8099만 보여도 시스템 설정에 따라 IPv4 연결을 함께 받을 수 있습니다.
```

7. 로컬 접속 확인
`curl -I http://127.0.0.1:8099/`
결과별 의미:
- 연결 거부: Apache가 8099에서 듣고 있지 않음
- 403: Apache는 정상이나 Require ip에서 차단
- 503: Apache는 정상이나 Tomcat 9910 연결 실패
- 200 또는 302: 정상
방화벽은 외부 접속에만 영향을 주므로 ss에 8099가 없는 문제와는 무관합니다. 우선 Listen 8099, httpd -t, SELinux 포트 등록, journalctl -u httpd 순으로 확인하면 됩니다.

---

> ```
> 10월 15 01:00:59 localhost.localdomain systemd[1]: httpd.service: Deactivated successfully.
> 10월 15 01:00:59 localhost.localdomain systemd[1]: Stopped The Apache HTTP Server.
> 10월 15 01:00:59 localhost.localdomain systemd[1]: httpd.service: Consumed 1min 11.534s CPU time, 24.8M memory peak.
> 10월 15 02:26:20 localhost.localdomain systemd[1]: Starting The Apache HTTP Server...
> 10월 15 02:26:20 localhost.localdomain httpd[232946]: Server configured, listening on: port 443, port 80
> 10월 15 02:26:20 localhost.localdomain systemd[1]: Started The Apache HTTP Server.
> 10월 15 02:32:24 localhost.localdomain systemd[1]: Stopping The Apache HTTP Server...
> 10월 15 02:32:25 localhost.localdomain systemd[1]: httpd.service: Deactivated successfully.
> 10월 15 02:32:25 localhost.localdomain systemd[1]: Stopped The Apache HTTP Server.
> 10월 15 02:32:25 localhost.localdomain systemd[1]: Starting The Apache HTTP Server...
> 10월 15 02:32:25 localhost.localdomain httpd[233242]: (13)Permission denied: AH00072: make_sock: could not bind to address [::]:8099
> 10월 15 02:32:25 localhost.localdomain httpd[233242]: (13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:8099
> 10월 15 02:32:25 localhost.localdomain httpd[233242]: no listening sockets available, shutting down
> 10월 15 02:32:25 localhost.localdomain httpd[233242]: AH00015: Unable to open logs
> 10월 15 02:32:25 localhost.localdomain systemd[1]: httpd.service: Main process exited, code=exited, status=1/FAILURE
> 10월 15 02:32:25 localhost.localdomain systemd[1]: httpd.service: Failed with result 'exit-code'.
> 10월 15 02:32:25 localhost.localdomain systemd[1]: Failed to start The Apache HTTP Server.
> ```

원인은 SELinux가 Apache의 8099 포트 바인딩을 차단한 것입니다.

핵심 오류:
`(13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:8099`
Apache 설정 문법이나 파일 권한보다는 8099 포트가 http_port_t로 등록되지 않은 것이 원인입니다.

1. SELinux 상태 확인
```bash
getenforce
```

2. Enforcing이면 아래 설정이 필요합니다.
```bash
# 현재 포트 타입 확인:
sudo semanage port -l | grep -w 8099
```

3. 8099를 Apache 포트로 등록
```bash
# 아무 결과가 없다면:
sudo semanage port -a \
  -t http_port_t \
  -p tcp \
  8099

# 등록 확인:
sudo semanage port -l \
  | grep http_port_t \
  | grep -w 8099
```

4. 이미 다른 타입으로 등록된 경우
```bash
# 다음 명령에서 다른 SELinux 타입이 나온다면:
sudo semanage port -l | grep -w 8099

# 8099를 사용하는 다른 서비스가 없는지 확인합니다.
sudo ss -lntp | grep ':8099'

# 다른 서비스가 없고 Apache 전용으로 사용할 경우 변경합니다.
sudo semanage port -m \
  -t http_port_t \
  -p tcp \
  8099
```

5. semanage가 없는 경우
```bash
sudo dnf install policycoreutils-python-utils
```
설치 후 포트를 등록합니다.

6. Apache가 Tomcat 9910에 접속하도록 허용
8099 바인딩 허용과는 별개의 설정입니다.
```bash
sudo setsebool -P \
  httpd_can_network_connect \
  1

# 확인:
getsebool httpd_can_network_connect

# 예상 결과:
httpd_can_network_connect --> on
```

7. Apache 설정 검사 및 시작
```bash
sudo httpd -t

# Syntax OK가 나오면:
sudo systemctl start httpd
sudo systemctl status httpd --no-pager

# 포트 확인:
sudo ss -lntp | grep ':8099'

# 정상 예:
0.0.0.0:8099

# 또는:
[::]:8099
```

8. 방화벽 설정 
서버 외부에서 8099로 접속해야 한다면:
```bash
sudo firewall-cmd --permanent \
  --add-port=8099/tcp

sudo firewall-cmd --reload

# 확인:
sudo firewall-cmd \
  --query-port=8099/tcp
```

9. 최종 확인
```bash
# Tomcat 직접 접속:
curl -I \
  http://127.0.0.1:9910/tsinspec/auth/login.do

# Apache 경유:
curl -I \
  http://127.0.0.1:8099/tsinspec/auth/login.do
```
로그의 다음 메시지는 8099 바인딩 실패에 이어 발생한 부수적인 종료 메시지일 가능성이 큽니다.
`AH00015: Unable to open logs`
8099를 http_port_t로 등록한 후에도 AH00015가 계속 나오면 그때 /var/log/httpd 권한과 SELinux 컨텍스트를 별도로 확인하면 됩니다.
SELinux를 setenforce 0으로 끄거나 Apache를 임의로 root 권한 확대하여 해결하지 말고, 8099 포트를 http_port_t로 등록하는 것이 올바른 조치입니다.
