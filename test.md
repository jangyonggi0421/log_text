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
직접 접속이 정상이면 Apache의 ProxyPass를 9910으로 변경합니다.
ProxyPass        "/tsinspec/" "http://127.0.0.1:18080/tsinspec/"
ProxyPassReverse "/tsinspec/" "http://127.0.0.1:18080/tsinspec/"
설정 검사:
sudo httpd -t
정상이면:
sudo systemctl start httpd
최종 접속:
curl -I \
  http://127.0.0.1/tsinspec/auth/login.do
11. 롤백 방법
신규 Tomcat에서 문제가 발생하면:
sudo systemctl stop httpd
sudo systemctl stop tsinspec-tomcat
Apache ProxyPass를 기존 Tomcat 포트로 되돌립니다.
sudo httpd -t
sudo systemctl start <기존_TOMCAT_서비스명>
sudo systemctl start httpd
DB를 변경하지 않았기 때문에 Tomcat과 Apache 설정만 되돌리면 됩니다.
테스트 완료 후에도 유지할 항목
정상 확인 후에도 바로 삭제하지 않습니다.
- 기존 Tomcat 설치 디렉터리
- 기존 Tomcat 서비스 파일
- 기존 Apache 설정 백업
- 기존 WAR
- 기존 업로드 파일
109번에서 며칠간 정상 운영을 확인한 뒤 신규 VM에도 동일한 /data/tsinspec 구조를 적용하면 됩니다. PostgreSQL과 DNF 설치 패키지는 이번 테스트 범위에서 그대로 유지하면 됩니다.


오전 12:32







DB 경로도 /data 하위로 모두 변경해줄래?







오전 12:41

## DB 경로 확인
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
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "SHOW port;"

sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "SHOW data_checksums;"
2. 외부 테이블스페이스와 WAL 경로 확인
기본 경로 밖의 테이블스페이스가 있는지 확인합니다.
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
pg_default, pg_global만 있고 location이 비어 있으면 일반적인 구성입니다.
WAL이 외부 경로로 연결됐는지도 확인합니다.
sudo ls -ld \
  /var/lib/pgsql/18/data/pg_wal

sudo readlink -f \
  /var/lib/pgsql/18/data/pg_wal
다음처럼 데이터 디렉터리 내부 경로이면 그대로 복사됩니다.
/var/lib/pgsql/18/data/pg_wal
별도 경로가 나온다면 해당 WAL 경로도 별도로 /data로 이전해야 하므로 바로 복사하지 말고 경로를 먼저 정리해야 합니다.
3. 이전 전 논리 백업
Tomcat을 중지하여 DB 쓰기를 차단합니다.
sudo systemctl stop tsinspec-tomcat
기존 서비스 이름을 아직 사용한다면 해당 서비스를 중지합니다.
sudo systemctl stop tomcat
DB는 실행 중인 상태에서 백업 디렉터리를 만듭니다.
sudo install -d \
  -o postgres \
  -g postgres \
  -m 700 \
  /data/postgresql/18/backup
애플리케이션 DB 백업:
sudo -u postgres /usr/pgsql-18/bin/pg_dump \
  -p 32419 \
  -d ts_insp_db \
  --format=custom \
  --compress=6 \
  --verbose \
  --file=/data/postgresql/18/backup/ts_insp_db_before_pgdata_move_20261015.dump
PostgreSQL 역할 백업:
sudo -u postgres /usr/pgsql-18/bin/pg_dumpall \
  -p 32419 \
  --globals-only \
  --file=/data/postgresql/18/backup/globals_before_pgdata_move_20261015.sql
역할 백업에는 PostgreSQL 비밀번호 검증값이 들어갈 수 있으므로 권한을 제한합니다.
sudo chmod 600 \
  /data/postgresql/18/backup/ts_insp_db_before_pgdata_move_20261015.dump \
  /data/postgresql/18/backup/globals_before_pgdata_move_20261015.sql
체크섬:
cd /data/postgresql/18/backup

sudo -u postgres sha256sum \
  ts_insp_db_before_pgdata_move_20261015.dump \
  globals_before_pgdata_move_20261015.sql \
  | sudo tee pgdata_move_20261015.sha256 >/dev/null
4. 이전 전 데이터 건수 기록
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
결과를 별도로 기록합니다.
5. 신규 데이터 경로 생성
먼저 기존 상태를 확인합니다.
sudo ls -la \
  /data/postgresql/18/data
디렉터리가 없으면 생성합니다.
sudo install -d \
  -o postgres \
  -g postgres \
  -m 700 \
  /data/postgresql/18/data
이미 다른 PostgreSQL 클러스터가 초기화되어 있다면 기존 데이터와 섞으면 안 됩니다. 별도로 보관합니다.
sudo mv \
  /data/postgresql/18/data \
  /data/postgresql/18/data-before-migration-20261015
그다음 빈 디렉터리를 다시 만듭니다.
sudo install -d \
  -o postgres \
  -g postgres \
  -m 700 \
  /data/postgresql/18/data
6. SELinux 경로 등록
sudo semanage fcontext -a \
  -t postgresql_db_t \
  '/data/postgresql/18(/.*)?'
이미 등록된 규칙이라는 오류가 나오면:
sudo semanage fcontext -m \
  -t postgresql_db_t \
  '/data/postgresql/18(/.*)?'
적용:
sudo restorecon -RFv \
  /data/postgresql/18
확인:
ls -Zd \
  /data/postgresql/18 \
  /data/postgresql/18/data
semanage가 없다면 먼저 설치합니다.
sudo dnf install policycoreutils-python-utils
7. PostgreSQL 완전 중지
sudo systemctl stop postgresql-18
상태 확인:
sudo systemctl status \
  postgresql-18 \
  --no-pager
프로세스가 남아 있지 않은지 확인합니다.
ps -ef | grep '[p]ostgres'
결과가 없어야 합니다.
8. 기존 데이터 디렉터리 복사
rsync가 있는지 확인합니다.
command -v rsync
없으면 설치합니다.
sudo dnf install rsync
기존 데이터 디렉터리가 /var/lib/pgsql/18/data인 경우:
sudo rsync \
  -aHAX \
  --numeric-ids \
  /var/lib/pgsql/18/data/ \
  /data/postgresql/18/data/
소유권과 권한을 다시 정리합니다.
sudo chown -R postgres:postgres \
  /data/postgresql/18/data

sudo chmod 700 \
  /data/postgresql/18/data
디스크에 기록을 반영합니다.
sync
두 디렉터리의 크기를 비교합니다.
sudo du -sh \
  /var/lib/pgsql/18/data \
  /data/postgresql/18/data
9. 새 데이터 디렉터리 검사
PostgreSQL 버전 정보 확인:
sudo -u postgres /usr/pgsql-18/bin/pg_controldata \
  /data/postgresql/18/data \
  | grep -E \
  'Database cluster state|Database block size|Data page checksum version'
정상 종료 상태는 다음과 같습니다.
Database cluster state: shut down
in production 또는 in crash recovery라면 서비스를 시작하지 말고 복사 과정을 다시 확인해야 합니다.
10. systemd의 PGDATA 변경
직접 서비스 원본 파일을 수정하지 않고 drop-in을 생성합니다.
sudo systemctl edit postgresql-18
다음 내용을 입력합니다.
[Service]
Environment=PGDATA=/data/postgresql/18/data
적용:
sudo systemctl daemon-reload
확인:
sudo systemctl show \
  postgresql-18 \
  --property=Environment
다음 값이 포함되어야 합니다.
PGDATA=/data/postgresql/18/data
11. PostgreSQL 시작
sudo systemctl start postgresql-18
상태 확인:
sudo systemctl status \
  postgresql-18 \
  --no-pager
문제가 발생하면:
sudo journalctl \
  -u postgresql-18 \
  -n 300 \
  --no-pager
12. 변경 결과 확인
데이터 경로:
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "SHOW data_directory;"
정상 결과:
/data/postgresql/18/data
포트:
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "SHOW port;"
DB 및 역할:
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "\l+"

sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "\du+"
테이블 확인:
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d ts_insp_db \
  -c "\dt inspection.*"
앞서 기록한 데이터 건수도 다시 비교합니다.
13. 설정 내 기존 경로 확인
새 데이터 디렉터리 안에 기존 /var/lib/pgsql 경로가 남아 있는지 확인합니다.
sudo grep -Rni \
  '/var/lib/pgsql' \
  /data/postgresql/18/data \
  --exclude='postmaster.pid'
다음 설정에서 기존 경로가 발견되면 /data 기준으로 수정합니다.
- archive_command
- restore_command
- log_directory
- 외부 SSL 인증서 경로
- 사용자 정의 include 파일
- 테이블스페이스 경로
WAL 아카이브를 사용할 경우:
archive_mode = on
archive_command = 'test ! -f /data/postgresql/18/archive/%f && cp %p /data/postgresql/18/archive/%f'
아카이브 디렉터리:
sudo install -d \
  -o postgres \
  -g postgres \
  -m 700 \
  /data/postgresql/18/archive

sudo restorecon -RFv \
  /data/postgresql/18/archive
WAL 아카이브를 사용하지 않는다면 archive_mode=off 상태로 유지합니다.
14. Tomcat 다시 시작
DB 확인이 끝나면 /data 기반 Tomcat을 시작합니다.
sudo systemctl start tsinspec-tomcat
확인:
sudo journalctl \
  -u tsinspec-tomcat \
  -n 300 \
  --no-pager
직접 접속:
curl -I \
  http://127.0.0.1:18080/tsinspec/auth/login.do
롤백 방법
신규 PGDATA에서 문제가 발생하면 원본 데이터 디렉터리를 삭제하지 않았으므로 되돌릴 수 있습니다.
PostgreSQL 중지:
sudo systemctl stop postgresql-18
현재 override 파일을 비활성화합니다.
sudo mv \
  /etc/systemd/system/postgresql-18.service.d/override.conf \
  /etc/systemd/system/postgresql-18.service.d/override.conf.disabled
적용:
sudo systemctl daemon-reload
sudo systemctl start postgresql-18
확인:
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "SHOW data_directory;"
기존 경로가 나오면 롤백된 것입니다.
삭제하지 말아야 할 경로
테스트가 끝나도 바로 삭제하지 않습니다.
/var/lib/pgsql/18/data
/data/postgresql/18/data-before-migration-20261015
/data/postgresql/18/backup
109번에서 /data 기반 Tomcat·PostgreSQL이 정상 동작하고 로그인, 자산, 점검, 보고서 기능까지 검증한 후에 기존 PostgreSQL 경로 정리를 검토하면 됩니다.

tsinspec-tomcat.service: Failed at step EXEC spawning /data/tsinspec/apps/tomcat/bin/catalina.sh: Permission denied 왜 이러지?







오전 1:37






14초 동안 작업함






이 오류는 Java나 Tomcat 설정을 읽기 전, systemd가 catalina.sh 자체를 실행하지 못한 상태입니다. 주요 원인은 다음 셋 중 하나입니다.
1. catalina.sh 실행 권한 누락
2. /data가 noexec로 마운트됨
3. SELinux가 /data의 실행을 차단함
순서대로 확인하면 됩니다.
1. 실행 권한과 상위 경로 확인
readlink -f \
  /data/tsinspec/apps/tomcat
예상 결과:
/data/tsinspec/apps/apache-tomcat-10.1.60
스크립트 권한 확인:
ls -lZ \
  /data/tsinspec/apps/tomcat/bin/catalina.sh
전체 상위 경로 권한 확인:
namei -l \
  /data/tsinspec/apps/tomcat/bin/catalina.sh
tomcat 계정이 실행할 수 있는지 확인합니다.
sudo -u tomcat test \
  -x /data/tsinspec/apps/tomcat/bin/catalina.sh

echo $?
- 0: 권한상 실행 가능
- 1: 실행 권한 또는 상위 디렉터리 접근 권한 문제
2. Tomcat 권한 재설정
sudo chown -R root:tomcat \
  /data/tsinspec/apps/apache-tomcat-10.1.60
디렉터리 권한:
sudo find \
  /data/tsinspec/apps/apache-tomcat-10.1.60 \
  -type d \
  -exec chmod 750 {} \;
일반 파일 권한:
sudo find \
  /data/tsinspec/apps/apache-tomcat-10.1.60 \
  -type f \
  -exec chmod 640 {} \;
스크립트에 실행 권한을 다시 부여합니다.
sudo chmod 750 \
  /data/tsinspec/apps/apache-tomcat-10.1.60/bin/*.sh
상위 디렉터리도 접근 가능하게 설정합니다.
sudo chown root:tomcat \
  /data/tsinspec \
  /data/tsinspec/apps

sudo chmod 750 \
  /data/tsinspec \
  /data/tsinspec/apps
/data 자체 권한도 확인합니다.
ls -ld /data
tomcat 계정에 /data 통과 권한이 없으면 ACL로 최소 권한만 부여할 수 있습니다.
sudo setfacl \
  -m u:tomcat:--x \
  /data
다시 확인합니다.
sudo -u tomcat test \
  -x /data/tsinspec/apps/tomcat/bin/catalina.sh

echo $?
3. /data의 noexec 확인
이번 오류에서는 반드시 확인해야 합니다.
findmnt \
  -no TARGET,SOURCE,FSTYPE,OPTIONS \
  /data
결과에 다음 옵션이 있다면 원인입니다.
noexec
예:
/data /dev/mapper/data xfs rw,nodev,nosuid,noexec
임시로 실행을 허용하려면:
sudo mount \
  -o remount,exec \
  /data
확인:
findmnt -no OPTIONS /data
재부팅 후에도 유지하려면 /etc/fstab의 /data 항목에서 noexec를 제거하고 exec 또는 기본값을 적용해야 합니다.
sudo vi /etc/fstab
예:
defaults,nodev,nosuid
보안 정책상 /data에 반드시 noexec가 있어야 한다면 Tomcat 프로그램은 /opt/apache-tomcat-10.1.60에 두고 다음 데이터만 /data에 두어야 합니다.
/data/tsinspec/tomcat-base
/data/tsinspec/data
/data/tsinspec/deploy
/data/tsinspec/backup
ExecStart=/bin/bash catalina.sh run처럼 noexec를 우회하는 방식은 운영 구성으로 권장하지 않습니다.
4. SELinux 확인
getenforce
Enforcing이면 최근 차단 기록을 확인합니다.
sudo ausearch \
  -m AVC \
  -ts recent \
  | tail -50
현재 컨텍스트 확인:
ls -ldZ \
  /data/tsinspec/apps \
  /data/tsinspec/apps/apache-tomcat-10.1.60

ls -lZ \
  /data/tsinspec/apps/tomcat/bin/catalina.sh
Tomcat 프로그램 경로를 실행 가능한 일반 프로그램 타입으로 영구 등록합니다.
sudo semanage fcontext -a \
  -t usr_t \
  '/data/tsinspec/apps(/.*)?'
규칙이 이미 존재한다면:
sudo semanage fcontext -m \
  -t usr_t \
  '/data/tsinspec/apps(/.*)?'
적용:
sudo restorecon -RFv \
  /data/tsinspec/apps
확인:
ls -lZ \
  /data/tsinspec/apps/tomcat/bin/catalina.sh
컨텍스트에 usr_t가 포함되어야 합니다.
5. 스크립트 형식 확인
file \
  /data/tsinspec/apps/tomcat/bin/catalina.sh
첫 줄 확인:
head -1 \
  /data/tsinspec/apps/tomcat/bin/catalina.sh
정상:
#!/bin/sh
Windows CRLF가 포함되면 다음 명령으로 변환합니다.
sudo sed -i 's/\r$//' \
  /data/tsinspec/apps/apache-tomcat-10.1.60/bin/*.sh
변환 후 실행 권한을 다시 적용합니다.
sudo chmod 750 \
  /data/tsinspec/apps/apache-tomcat-10.1.60/bin/*.sh
6. systemd 없이 직접 설정 검사
sudo -u tomcat env \
  JAVA_HOME=/usr/lib/jvm/java-21-openjdk \
  CATALINA_HOME=/data/tsinspec/apps/tomcat \
  CATALINA_BASE=/data/tsinspec/tomcat-base \
  /data/tsinspec/apps/tomcat/bin/catalina.sh configtest
이 명령이 성공해야 systemd에서도 실행할 수 있습니다.
7. 서비스 재시작
sudo systemctl daemon-reload
sudo systemctl restart tsinspec-tomcat
확인:
sudo systemctl status \
  tsinspec-tomcat \
  --no-pager
sudo journalctl \
  -u tsinspec-tomcat \
  -n 200 \
  --no-pager
가장 먼저 다음 두 결과를 확인하면 원인을 빠르게 구분할 수 있습니다.
sudo -u tomcat test \
  -x /data/tsinspec/apps/tomcat/bin/catalina.sh
echo $?

findmnt -no OPTIONS /data
실행 권한 결과가 0인데도 systemd에서 계속 Permission denied가 발생하면 /data의 noexec 또는 SELinux가 원인일 가능성이 가장 높습니다.


오전 1:37







findmnt \
  -no TARGET,SOURCE,FSTYPE,OPTIONS \
  /data
결과가 없는데?







오전 1:40






5초 동안 작업함






결과가 없다는 것은 /data가 독립적인 마운트 지점으로 등록되지 않았을 가능성이 큽니다. findmnt /data는 정확히 /data에 마운트된 파일시스템만 찾습니다.
아무것도 출력되지 않았다고 해서 문제가 있는 것은 아니며, 지금은 remount 명령을 실행하면 안 됩니다.
1. /data가 속한 실제 파일시스템 확인
-T 옵션으로 확인합니다.
findmnt \
  -T /data \
  -o TARGET,SOURCE,FSTYPE,OPTIONS
예를 들어 다음처럼 나오면:
TARGET SOURCE                 FSTYPE OPTIONS
/      /dev/mapper/rl-root    xfs    rw,relatime,seclabel
/data는 별도 마운트가 아니라 루트 / 파일시스템 안의 일반 디렉터리입니다.
다음처럼 나오면 별도 마운트입니다.
TARGET SOURCE                 FSTYPE OPTIONS
/data  /dev/mapper/rl-data    xfs    rw,relatime,seclabel
추가 확인:
mountpoint /data
echo $?
- 0: /data는 별도 마운트 지점
- 1: /data는 일반 디렉터리
용량 기준 확인:
df -hT /data
2. noexec 여부 판단
다음 결과에서 noexec가 있는지만 확인합니다.
findmnt \
  -T /data \
  -no OPTIONS
noexec가 없다면 마운트 문제는 제외하고 파일 권한과 SELinux를 확인하면 됩니다.
3. Tomcat 계정 실행 권한 확인
sudo -u tomcat test \
  -x /data/tsinspec/apps/tomcat/bin/catalina.sh

echo $?
결과가 1이면 권한 문제입니다.
경로 전체 권한을 확인합니다.
namei -l \
  /data/tsinspec/apps/tomcat/bin/catalina.sh
다음 모든 경로에 tomcat 사용자가 통과할 수 있는 x 권한이 있어야 합니다.
/data
/data/tsinspec
/data/tsinspec/apps
/data/tsinspec/apps/apache-tomcat-10.1.60
/data/tsinspec/apps/apache-tomcat-10.1.60/bin
catalina.sh
권장 권한을 다시 적용합니다.
sudo chown root:tomcat \
  /data/tsinspec \
  /data/tsinspec/apps
sudo chmod 750 \
  /data/tsinspec \
  /data/tsinspec/apps
sudo chown -R root:tomcat \
  /data/tsinspec/apps/apache-tomcat-10.1.60
sudo find \
  /data/tsinspec/apps/apache-tomcat-10.1.60 \
  -type d \
  -exec chmod 750 {} \;
sudo chmod 750 \
  /data/tsinspec/apps/apache-tomcat-10.1.60/bin/*.sh
/data 자체가 700 root:root라면 Tomcat이 통과할 수 없습니다.
ls -ld /data
이 경우 전체 공개 대신 Tomcat 계정에 통과 권한만 줄 수 있습니다.
sudo setfacl \
  -m u:tomcat:--x \
  /data
4. SELinux 확인
권한 검사 결과가 0인데도 실행이 거부되면 SELinux를 확인합니다.
getenforce
ls -lZ \
  /data/tsinspec/apps/tomcat/bin/catalina.sh
sudo ausearch \
  -m AVC \
  -ts recent \
  | tail -50
default_t 또는 SELinux 차단 기록이 확인되면:
sudo semanage fcontext -a \
  -t usr_t \
  '/data/tsinspec/apps(/.*)?'
이미 규칙이 있다면:
sudo semanage fcontext -m \
  -t usr_t \
  '/data/tsinspec/apps(/.*)?'
적용:
sudo restorecon -RFv \
  /data/tsinspec/apps
5. 직접 실행 검사
sudo -u tomcat env \
  JAVA_HOME=/usr/lib/jvm/java-21-openjdk \
  CATALINA_HOME=/data/tsinspec/apps/tomcat \
  CATALINA_BASE=/data/tsinspec/tomcat-base \
  /data/tsinspec/apps/tomcat/bin/catalina.sh configtest
이 명령이 성공하면 서비스를 다시 시작합니다.
sudo systemctl restart tsinspec-tomcat
sudo systemctl status tsinspec-tomcat --no-pager
현재는 우선 다음 세 명령 결과가 핵심입니다.
findmnt -T /data -o TARGET,SOURCE,FSTYPE,OPTIONS
namei -l /data/tsinspec/apps/tomcat/bin/catalina.sh
sudo -u tomcat test -x /data/tsinspec/apps/tomcat/bin/catalina.sh; echo $?
findmnt -T에 noexec가 없고 마지막 결과가 1이면 상위 디렉터리 또는 catalina.sh 실행 권한 문제입니다. 마지막 결과가 0인데 systemd만 실패하면 SELinux 차단 여부를 확인해야 합니다.


오전 1:40







일반 디렉터리면 실행 불가인가?







오전 1:42






아니요. /data가 일반 디렉터리여도 Tomcat을 정상적으로 실행할 수 있습니다.
중요한 것은 일반 디렉터리인지 별도 마운트인지가 아니라 다음 조건입니다.
- 파일시스템에 noexec 옵션이 없음
- tomcat 계정이 상위 디렉터리를 통과할 수 있음
- catalina.sh에 실행 권한이 있음
- SELinux가 실행을 차단하지 않음
1. 파일시스템 실행 가능 여부
findmnt \
  -T /data \
  -o TARGET,SOURCE,FSTYPE,OPTIONS
결과에 noexec가 없으면 파일시스템에서 실행할 수 있습니다.
rw,relatime,seclabel
위와 같은 결과는 정상입니다.
2. 상위 디렉터리 접근 권한
namei -l \
  /data/tsinspec/apps/tomcat/bin/catalina.sh
Tomcat 계정이 다음 모든 디렉터리를 통과할 수 있어야 합니다.
/data
/data/tsinspec
/data/tsinspec/apps
/data/tsinspec/apps/apache-tomcat-10.1.60
/data/tsinspec/apps/apache-tomcat-10.1.60/bin
/data가 다음과 같으면 Tomcat이 접근할 수 없습니다.
drwx------ root root /data
이 경우 Tomcat에 통과 권한만 부여합니다.
sudo setfacl \
  -m u:tomcat:--x \
  /data
또는 /data가 공용 애플리케이션 경로라면:
sudo chmod 755 /data
ACL 방식이 더 제한적입니다.
3. 실행파일 권한
ls -l \
  /data/tsinspec/apps/tomcat/bin/catalina.sh
정상 예:
-rwxr-x--- root tomcat catalina.sh
다르면 수정합니다.
sudo chown root:tomcat \
  /data/tsinspec/apps/apache-tomcat-10.1.60/bin/catalina.sh

sudo chmod 750 \
  /data/tsinspec/apps/apache-tomcat-10.1.60/bin/catalina.sh
다른 Tomcat 스크립트도 적용합니다.
sudo chmod 750 \
  /data/tsinspec/apps/apache-tomcat-10.1.60/bin/*.sh
4. Tomcat 계정으로 직접 확인
sudo -u tomcat test \
  -x /data/tsinspec/apps/tomcat/bin/catalina.sh

echo $?
- 0: 리눅스 파일 권한상 실행 가능
- 1: 실행 권한 또는 상위 경로 접근 권한 문제
직접 실행:
sudo -u tomcat \
  /data/tsinspec/apps/tomcat/bin/catalina.sh version
이 명령이 성공하면 일반 디렉터리인 것은 문제가 아닙니다.
5. SELinux 확인
파일 권한 검사가 0인데도 Permission denied라면 SELinux 가능성이 큽니다.
getenforce
ls -lZ \
  /data/tsinspec/apps/tomcat/bin/catalina.sh
default_t로 표시되면 컨텍스트를 등록합니다.
sudo semanage fcontext -a \
  -t usr_t \
  '/data/tsinspec/apps(/.*)?'
이미 등록되어 있다면:
sudo semanage fcontext -m \
  -t usr_t \
  '/data/tsinspec/apps(/.*)?'
적용:
sudo restorecon -RFv \
  /data/tsinspec/apps
그런 다음 다시 시작합니다.
sudo systemctl restart tsinspec-tomcat
sudo systemctl status tsinspec-tomcat --no-pager
따라서 /data가 일반 디렉터리인 것은 실행 오류의 원인이 아닙니다. 현재 오류는 파일·상위 디렉터리 권한 또는 SELinux 컨텍스트일 가능성이 높습니다.


오전 1:42







rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota 결과면?







오전 1:42






5초 동안 작업함






해당 결과에는 noexec가 없으므로 /data 파일시스템에서 실행할 수 있습니다.
rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
따라서 마운트 옵션은 원인이 아닙니다. seclabel이 있으므로 이제 파일 권한과 SELinux를 확인해야 합니다.
1. 실행 권한 확인
sudo -u tomcat test \
  -x /data/tsinspec/apps/tomcat/bin/catalina.sh

echo $?
결과가 1
파일 또는 상위 디렉터리 권한 문제입니다.
namei -l \
  /data/tsinspec/apps/tomcat/bin/catalina.sh
권한을 다시 설정합니다.
sudo chown -R root:tomcat \
  /data/tsinspec/apps/apache-tomcat-10.1.60
sudo find \
  /data/tsinspec/apps/apache-tomcat-10.1.60 \
  -type d \
  -exec chmod 750 {} \;
sudo chmod 750 \
  /data/tsinspec/apps/apache-tomcat-10.1.60/bin/*.sh
상위 경로:
sudo chown root:tomcat \
  /data/tsinspec \
  /data/tsinspec/apps

sudo chmod 750 \
  /data/tsinspec \
  /data/tsinspec/apps
/data 자체도 확인합니다.
ls -ld /data
tomcat이 통과할 수 없다면:
sudo setfacl \
  -m u:tomcat:--x \
  /data
결과가 0
일반 파일 권한은 정상입니다. SELinux 차단 가능성이 큽니다.
2. SELinux 상태와 컨텍스트 확인
getenforce
ls -ldZ \
  /data \
  /data/tsinspec \
  /data/tsinspec/apps \
  /data/tsinspec/apps/apache-tomcat-10.1.60
ls -lZ \
  /data/tsinspec/apps/tomcat/bin/catalina.sh
default_t로 표시되면 Tomcat 프로그램 경로를 등록합니다.
sudo semanage fcontext -a \
  -t usr_t \
  '/data/tsinspec/apps(/.*)?'
이미 등록된 규칙이라는 오류가 나오면:
sudo semanage fcontext -m \
  -t usr_t \
  '/data/tsinspec/apps(/.*)?'
적용:
sudo restorecon -RFv \
  /data/tsinspec/apps
다시 확인합니다.
ls -lZ \
  /data/tsinspec/apps/tomcat/bin/catalina.sh
컨텍스트에 usr_t가 포함되어야 합니다.
3. SELinux 차단 로그 확인
sudo ausearch \
  -m AVC \
  -ts recent \
  | tail -50
catalina.sh, /data/tsinspec/apps 또는 execute 관련 denied가 있으면 SELinux가 원인입니다.
4. Tomcat 계정으로 직접 실행
sudo -u tomcat env \
  JAVA_HOME=/usr/lib/jvm/java-21-openjdk \
  CATALINA_HOME=/data/tsinspec/apps/tomcat \
  CATALINA_BASE=/data/tsinspec/tomcat-base \
  /data/tsinspec/apps/tomcat/bin/catalina.sh version
정상적으로 버전이 출력되면 설정 검사도 실행합니다.
sudo -u tomcat env \
  JAVA_HOME=/usr/lib/jvm/java-21-openjdk \
  CATALINA_HOME=/data/tsinspec/apps/tomcat \
  CATALINA_BASE=/data/tsinspec/tomcat-base \
  /data/tsinspec/apps/tomcat/bin/catalina.sh configtest
5. 서비스 재시작
sudo systemctl restart tsinspec-tomcat
sudo systemctl status \
  tsinspec-tomcat \
  --no-pager
sudo journalctl \
  -u tsinspec-tomcat \
  -n 200 \
  --no-pager
현재 마운트 옵션은 정상입니다. 가장 가능성이 높은 원인은 catalina.sh 실행 비트 누락 또는 /data에 적용된 SELinux 컨텍스트입니다.


오전 1:42







10월 15 01:43:39 localhost.localdomain systemd[1]: tsinspec-tomcat.service: Main process exited, code=exited, status=203/EXEC
10월 15 01:43:39 localhost.localdomain systemd[1]: tsinspec-tomcat.service: Failed with result 'exit-code'.
10월 15 01:43:49 localhost.localdomain systemd[1]: tsinspec-tomcat.service: Scheduled restart job, restart counter is at 88.
10월 15 01:43:49 localhost.localdomain systemd[1]: Stopped TS Inspection Tomcat 10.1.60.
10월 15 01:43:49 localhost.localdomain systemd[1]: Started TS Inspection Tomcat 10.1.60.
10월 15 01:43:49 localhost.localdomain systemd[231636]: tsinspec-tomcat.service: Failed to locate executable /data/tsinspec/apps/tomcat/bin/catalina.sh: Permission denied
10월 15 01:43:49 localhost.localdomain systemd[231636]: tsinspec-tomcat.service: Failed at step EXEC spawning /data/tsinspec/apps/tomcat/bin/catalina.sh: Permission denied
10월 15 01:43:49 localhost.localdomain systemd[1]: tsinspec-tomcat.service: Main process exited, code=exited, status=203/EXEC
10월 15 01:43:49 localhost.localdomain systemd[1]: tsinspec-tomcat.service: Failed with result 'exit-code'.
10월 15 01:43:59 localhost.localdomain systemd[1]: tsinspec-tomcat.service: Scheduled restart job, restart counter is at 89.
10월 15 01:43:59 localhost.localdomain systemd[1]: Stopped TS Inspection Tomcat 10.1.60.
10월 15 01:43:59 localhost.localdomain systemd[1]: Started TS Inspection Tomcat 10.1.60.
10월 15 01:43:59 localhost.localdomain catalina.sh[231646]: The JAVA_HOME environment variable is not defined correctly
10월 15 01:43:59 localhost.localdomain catalina.sh[231646]: JAVA_HOME=/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64/bin/java
10월 15 01:43:59 localhost.localdomain catalina.sh[231646]: This environment variable is needed to run this program
10월 15 01:43:59 localhost.localdomain catalina.sh[231646]: NB: JAVA_HOME should point to a JDK not a JRE
10월 15 01:43:59 localhost.localdomain systemd[1]: tsinspec-tomcat.service: Main process exited, code=exited, status=1/FAILURE
10월 15 01:43:59 localhost.localdomain systemd[1]: tsinspec-tomcat.service: Failed with result 'exit-code'.
10월 15 01:44:09 localhost.localdomain systemd[1]: tsinspec-tomcat.service: Scheduled restart job, restart counter is at 90.
10월 15 01:44:09 localhost.localdomain systemd[1]: Stopped TS Inspection Tomcat 10.1.60.
10월 15 01:44:09 localhost.localdomain systemd[1]: Started TS Inspection Tomcat 10.1.60.
10월 15 01:44:09 localhost.localdomain catalina.sh[231651]: The JAVA_HOME environment variable is not defined correctly
10월 15 01:44:09 localhost.localdomain catalina.sh[231651]: JAVA_HOME=/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64/bin/java
10월 15 01:44:09 localhost.localdomain catalina.sh[231651]: This environment variable is needed to run this program
10월 15 01:44:09 localhost.localdomain catalina.sh[231651]: NB: JAVA_HOME should point to a JDK not a JRE
10월 15 01:44:09 localhost.localdomain systemd[1]: tsinspec-tomcat.service: Main process exited, code=exited, status=1/FAILURE
10월 15 01:44:09 localhost.localdomain systemd[1]: tsinspec-tomcat.service: Failed with result 'exit-code'.
10월 15 01:44:09 localhost.localdomain systemd[1]: Stopped TS Inspection Tomcat 10.1.60.
10월 15 01:44:09 localhost.localdomain systemd[1]: Started TS Inspection Tomcat 10.1.60.
10월 15 01:44:09 localhost.localdomain catalina.sh[231661]: The JAVA_HOME environment variable is not defined correctly
10월 15 01:44:09 localhost.localdomain catalina.sh[231661]: JAVA_HOME=/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64/bin/java
10월 15 01:44:09 localhost.localdomain catalina.sh[231661]: This environment variable is needed to run this program
10월 15 01:44:09 localhost.localdomain catalina.sh[231661]: NB: JAVA_HOME should point to a JDK not a JRE
10월 15 01:44:09 localhost.localdomain systemd[1]: tsinspec-tomcat.service: Main process exited, code=exited, status=1/FAILURE
10월 15 01:44:09 localhost.localdomain systemd[1]: tsinspec-tomcat.service: Failed with result 'exit-code'.
이런데 Apache 설정과는 별도지?







오전 1:45






9초 동안 작업함






네. 현재 오류는 Apache 설정과 완전히 별개입니다. Apache 요청이 Tomcat까지 전달되기 전에 Tomcat 프로세스 자체가 시작되지 못하고 있습니다.
로그를 보면 두 단계로 상황이 바뀌었습니다.
1. 처음에는 catalina.sh 실행 권한 오류
2. 이후에는 catalina.sh가 실행되었지만 JAVA_HOME이 잘못 설정됨
현재 핵심 오류는 다음입니다.
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64/bin/java
JAVA_HOME에는 java 실행파일이 아니라 JDK 설치 디렉터리가 들어가야 합니다.
올바른 형태:
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64
잘못된 형태:
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64/bin/java
1. 반복 재시작 중지
현재 재시작 횟수가 90회이므로 먼저 서비스를 중지합니다.
sudo systemctl stop tsinspec-tomcat
sudo systemctl reset-failed tsinspec-tomcat
확인:
sudo systemctl status \
  tsinspec-tomcat \
  --no-pager
2. 실제 JDK 경로 확인
command -v java
command -v javac
readlink -f "$(command -v java)"
readlink -f "$(command -v javac)"
javac 결과가 다음과 같다고 가정하면:
/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64/bin/javac
JAVA_HOME은 다음입니다.
/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64
자동으로 계산하여 확인할 수도 있습니다.
dirname \
  "$(dirname "$(readlink -f "$(command -v javac)")")"
3. javac가 없는 경우
command -v javac
결과가 없다면 JRE만 설치되어 있을 가능성이 있습니다.
rpm -qa \
  | grep -E '^java-21-openjdk'
JDK 개발 패키지가 없으면 설치합니다.
sudo dnf install java-21-openjdk-devel
설치 후 확인:
java -version
javac -version
4. 잘못된 JAVA_HOME 설정 위치 찾기
다음 위치를 모두 확인합니다.
sudo grep -Rns \
  'JAVA_HOME' \
  /etc/systemd/system/tsinspec-tomcat.service \
  /etc/systemd/system/tsinspec-tomcat.service.d \
  /etc/sysconfig/tsinspec-tomcat \
  /data/tsinspec/tomcat-base/bin \
  2>/dev/null
다음처럼 /bin/java까지 설정된 부분을 찾아야 합니다.
JAVA_HOME=/usr/lib/jvm/.../bin/java
5. systemd 서비스 수정
sudo vi \
  /etc/systemd/system/tsinspec-tomcat.service
잘못된 설정:
Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64/bin/java"
올바른 설정:
Environment="JAVA_HOME=/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64"
실제 javac 경로에서 /bin/javac을 제외한 디렉터리를 사용합니다.
JRE_HOME이 별도로 잘못 설정되어 있다면 제거하는 것이 좋습니다.
6. 환경변수 파일도 확인
sudo vi /etc/sysconfig/tsinspec-tomcat
이 파일에 JAVA_HOME이 있다면 동일하게 디렉터리까지만 지정합니다.
JAVA_HOME="/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64"
다만 JAVA_HOME은 서비스 파일 한 곳에서만 설정하는 것이 더 명확합니다. 서비스 파일에 설정했다면 /etc/sysconfig/tsinspec-tomcat에서는 JAVA_HOME을 제거하고 애플리케이션 환경변수만 유지하는 것을 권장합니다.
7. JDK 경로 검증
설정한 경로로 직접 확인합니다.
/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64/bin/java \
  -version
/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64/bin/javac \
  -version
두 명령이 모두 정상이어야 합니다.
8. Tomcat 계정으로 설정 검사
실제 JDK 경로를 사용합니다.
sudo -u tomcat env \
  JAVA_HOME=/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64 \
  CATALINA_HOME=/data/tsinspec/apps/tomcat \
  CATALINA_BASE=/data/tsinspec/tomcat-base \
  /data/tsinspec/apps/tomcat/bin/catalina.sh configtest
정상이라면 다음과 유사하게 출력됩니다.
Server configuration loaded successfully
9. systemd 반영 및 재시작
sudo systemctl daemon-reload
sudo systemctl reset-failed tsinspec-tomcat
sudo systemctl start tsinspec-tomcat
확인:
sudo systemctl status \
  tsinspec-tomcat \
  --no-pager
sudo journalctl \
  -u tsinspec-tomcat \
  -n 200 \
  --no-pager
포트 확인:
sudo ss -lntp \
  | grep 18080
10. Apache 연동 전 직접 확인
curl -I \
  http://127.0.0.1:18080/tsinspec/auth/login.do
여기서 Tomcat 직접 접속이 정상 확인된 뒤 Apache를 점검하면 됩니다.
정리하면, 처음의 203/EXEC 권한 문제는 이후 로그에서 catalina.sh가 실행된 것으로 보아 해결된 상태입니다. 현재 남은 원인은 JAVA_HOME 뒤에 /bin/java까지 들어간 것입니다. 이를 JDK 디렉터리까지만 지정하면 됩니다.

웹 미리보기웹사이트열기다음에서 열기






오전 1:45



/data/postgresql/18/data/log/postgresql-Wed.log:18:2026-10-14 12:04:33.336 KST [196601] 상세정보:  "/var/lib/pgsql/18/data/pg_hba.conf" 파일의 114번째 줄에 지정한 인증 설정이 사용됨: "local   all             all                                     scram-sha-256" 의미가?







오전 2:08






7초 동안 작업함






해당 로그는 PostgreSQL이 로컬 Unix 소켓 접속을 인증하면서 pg_hba.conf의 다음 규칙을 사용했다는 의미입니다.
local   all   all   scram-sha-256
각 항목의 의미는 다음과 같습니다.
- local: TCP/IP가 아닌 Unix 소켓 접속
- 첫 번째 all: 모든 데이터베이스
- 두 번째 all: 모든 PostgreSQL 사용자
- scram-sha-256: 비밀번호 기반 SCRAM 인증 요구
즉, 다음과 같은 접속에 적용됩니다.
psql -d ts_insp_db -U tsins_user
반면 다음처럼 127.0.0.1을 지정한 접속은 local이 아니라 host 규칙을 사용합니다.
psql \
  -h 127.0.0.1 \
  -p 32419 \
  -U tsins_user \
  -d ts_insp_db
오류 메시지인지 여부
해당 상세정보 한 줄 자체는 오류가 아닙니다. 보통 바로 앞에 다음과 같은 인증 실패 원인이 함께 기록됩니다.
FATAL: password authentication failed for user "..."
또는:
FATAL: no password supplied
앞뒤 로그를 같이 확인해야 합니다.
sudo sed -n '10,25p' \
  /data/postgresql/18/data/log/postgresql-Wed.log
이전 /var/lib 경로가 표시되는 이유
로그 시각은 다음입니다.
2026-10-14 12:04:33
현재 /data로 이동하면서 기존 로그 파일까지 복사했다면, 이 메시지는 이전 데이터 경로를 사용하던 시점의 과거 기록일 수 있습니다.
현재 PostgreSQL이 실제로 어떤 파일을 사용하는지 확인합니다.
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "SHOW data_directory;"
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "SHOW config_file;"
sudo -u postgres /usr/pgsql-18/bin/psql \
  -p 32419 \
  -d postgres \
  -c "SHOW hba_file;"
정상적으로 /data 전환이 완료됐다면 다음처럼 나와야 합니다.
data_directory | /data/postgresql/18/data
config_file    | /data/postgresql/18/data/postgresql.conf
hba_file       | /data/postgresql/18/data/pg_hba.conf
현재도 /var/lib가 나오는 경우
현재 SHOW hba_file 결과도 다음이라면:
/var/lib/pgsql/18/data/pg_hba.conf
PostgreSQL이 아직 기존 설정을 사용하고 있거나 hba_file이 절대경로로 지정된 것입니다.
현재 설정에서 확인합니다.
sudo grep -nE \
  '^[[:space:]]*(data_directory|hba_file|ident_file)' \
  /data/postgresql/18/data/postgresql.conf
다음처럼 절대경로가 지정되어 있으면 수정해야 합니다.
hba_file = '/var/lib/pgsql/18/data/pg_hba.conf'
권장:
hba_file = '/data/postgresql/18/data/pg_hba.conf'
또는 hba_file 설정을 주석 처리하면 기본적으로 현재 PGDATA 아래의 pg_hba.conf를 사용합니다.
변경 후에는 재시작합니다.
sudo systemctl restart postgresql-18
권장 로컬 인증 설정
/data/postgresql/18/data/pg_hba.conf에 다음처럼 구성할 수 있습니다.
# postgres OS 계정은 Unix 소켓 peer 인증
local   all           postgres                              peer

# 기타 로컬 Unix 소켓 접속은 비밀번호 인증
local   all           all                                   scram-sha-256

# 동일 VM의 Tomcat JDBC 접속
host    ts_insp_db    tsins_user    127.0.0.1/32            scram-sha-256
host    ts_insp_db    tsins_user    ::1/128                 scram-sha-256
이 구성에서는:
sudo -u postgres psql -p 32419 -d postgres
은 비밀번호 없이 접속할 수 있고, tsins_user는 SCRAM 비밀번호가 필요합니다.
Tomcat의 JDBC 주소가 다음이라면:
TSINSPEC_DB_URL="jdbc:postgresql://127.0.0.1:32419/ts_insp_db?currentSchema=inspection"
local 규칙이 아니라 다음 host 규칙이 적용됩니다.
host ts_insp_db tsins_user 127.0.0.1/32 scram-sha-256
따라서 우선 SHOW hba_file; 결과를 확인하면 과거 로그인지, 아직 기존 /var/lib 설정을 사용하는지 바로 구분할 수 있습니다.


오전 2:08


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


```conf
<VirtualHost *:8099>
    ServerName 192.168.196.109
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
```

톰캣 포트를 9910으로 수정한 뒤에 배포가 안된다면?







오전 2:16






4초 동안 작업함






Tomcat 포트를 9910으로 바꾼 것 자체는 WAR 배포에 영향을 주지 않습니다. 다만 다음 중 하나가 원인일 가능성이 큽니다.
- 실제 CATALINA_BASE가 아닌 다른 server.xml 수정
- 9910 포트 충돌
- server.xml XML 문법 오류
- Tomcat 재시작 누락
- Apache ProxyPass가 기존 포트를 계속 사용
- WAR 경로 또는 권한 문제
다음 순서로 확인하면 원인을 구분할 수 있습니다.
1. 실제 CATALINA_BASE 확인
sudo systemctl cat tsinspec-tomcat
다음 설정을 확인합니다.
Environment="CATALINA_BASE=/data/tsinspec/tomcat-base"
그러면 수정해야 하는 파일은 다음입니다.
/data/tsinspec/tomcat-base/conf/server.xml
Tomcat 원본 경로의 다음 파일을 수정해도 적용되지 않을 수 있습니다.
/data/tsinspec/apps/tomcat/conf/server.xml
2. Connector 설정 확인
sudo grep -nA8 -B2 \
  'port="9910"' \
  /data/tsinspec/tomcat-base/conf/server.xml
권장 설정:
<Connector
    address="127.0.0.1"
    port="9910"
    protocol="HTTP/1.1"
    connectionTimeout="20000"
    URIEncoding="UTF-8"
    maxParameterCount="1000"
    redirectPort="8443" />
Tomcat이 별도 VM의 Apache 요청을 받아야 하는 구조라면 127.0.0.1이 아닌 Tomcat 서버 내부 IP로 설정해야 합니다. Apache와 Tomcat이 같은 VM이면 127.0.0.1이 맞습니다.
3. 포트 충돌 확인
Tomcat을 중지한 상태에서 확인합니다.
sudo systemctl stop tsinspec-tomcat
sudo ss -lntp \
  | grep ':9910'
결과가 없어야 합니다. 다른 프로세스가 이미 9910 포트를 사용하고 있다면 Tomcat이 시작되지 않습니다.
4. Tomcat 설정 검사
sudo -u tomcat env \
  JAVA_HOME=<실제_JDK_디렉터리> \
  CATALINA_HOME=/data/tsinspec/apps/tomcat \
  CATALINA_BASE=/data/tsinspec/tomcat-base \
  /data/tsinspec/apps/tomcat/bin/catalina.sh configtest
예:
sudo -u tomcat env \
  JAVA_HOME=/usr/lib/jvm/java-21-openjdk-21.0.12.1.1-1.2.el9_8.x86_64 \
  CATALINA_HOME=/data/tsinspec/apps/tomcat \
  CATALINA_BASE=/data/tsinspec/tomcat-base \
  /data/tsinspec/apps/tomcat/bin/catalina.sh configtest
오류가 발생하면 server.xml 문법부터 수정해야 합니다.
5. WAR 위치와 권한 확인
sudo ls -lh \
  /data/tsinspec/tomcat-base/webapps/tsinspec.war
정상 권한 예:
-rw-r----- tomcat tomcat tsinspec.war
수정:
sudo chown tomcat:tomcat \
  /data/tsinspec/tomcat-base/webapps/tsinspec.war

sudo chmod 640 \
  /data/tsinspec/tomcat-base/webapps/tsinspec.war
WAR 내부 확인:
jar tf \
  /data/tsinspec/tomcat-base/webapps/tsinspec.war \
  | grep 'WEB-INF/web.xml'
6. 기존 압축 해제 디렉터리 정리
기존 tsinspec 디렉터리가 남아 있으면 예전 파일과 신규 WAR가 섞일 수 있습니다. 삭제하지 말고 백업 위치로 이동합니다.
TSINSPEC_DEPLOY_TIME=$(date +%Y%m%d_%H%M%S)
sudo mv \
  /data/tsinspec/tomcat-base/webapps/tsinspec \
  /data/tsinspec/backup/tsinspec-expanded-${TSINSPEC_DEPLOY_TIME}
tsinspec 디렉터리가 존재할 때만 실행합니다.
7. Tomcat 시작
sudo systemctl daemon-reload
sudo systemctl reset-failed tsinspec-tomcat
sudo systemctl start tsinspec-tomcat
상태 확인:
sudo systemctl status \
  tsinspec-tomcat \
  --no-pager
배포 로그:
sudo journalctl \
  -u tsinspec-tomcat \
  -n 500 \
  --no-pager
다음과 같은 메시지를 확인합니다.
Deploying web application archive
Deployment of web application archive
Server startup
8. 9910 포트 확인
sudo ss -lntp \
  | grep ':9910'
정상 예:
127.0.0.1:9910
기존 포트도 확인합니다.
sudo ss -lntp \
  | grep -E ':8080|:18080|:9910'
Tomcat이 여전히 기존 포트에서 실행된다면 수정한 server.xml이 실제 CATALINA_BASE 설정이 아닐 가능성이 큽니다.
9. Apache를 거치지 않고 직접 확인
curl -I \
  http://127.0.0.1:9910/tsinspec/auth/login.do
또는 상태 코드만 확인합니다.
curl -sS \
  -o /dev/null \
  -w 'HTTP %{http_code}\n' \
  http://127.0.0.1:9910/tsinspec/auth/login.do
결과별 의미:
결과	의미
200, 302	Tomcat과 애플리케이션 정상
연결 거부	Tomcat 미기동 또는 Connector 오류
404	WAR 미배포 또는 컨텍스트 경로 오류
500	애플리케이션 초기화·DB·Spring 오류
503	보통 Apache 프록시 또는 애플리케이션 서비스 오류


10. Apache ProxyPass 수정
Tomcat 직접 접속이 정상인 경우 Apache 포트를 수정합니다.
sudo vi /etc/httpd/conf.d/tsinspec.conf
기존:
ProxyPass        "/tsinspec/" "http://127.0.0.1:18080/tsinspec/"
ProxyPassReverse "/tsinspec/" "http://127.0.0.1:18080/tsinspec/"
변경:
ProxyPass        "/tsinspec/" \
                 "http://127.0.0.1:9910/tsinspec/" \
                 connectiontimeout=5 timeout=120

ProxyPassReverse "/tsinspec/" \
                 "http://127.0.0.1:9910/tsinspec/"
기존 포트가 다른 Apache 설정에 남아 있는지 확인합니다.
sudo grep -RniE \
  '8080|18080|9910|ProxyPass' \
  /etc/httpd/conf \
  /etc/httpd/conf.d
11. Apache SELinux 및 재시작
sudo setsebool -P \
  httpd_can_network_connect \
  1
설정 검사:
sudo httpd -t
Syntax OK인 경우에만 재시작합니다.
sudo systemctl restart httpd
sudo systemctl status httpd --no-pager
최종 확인:
curl -I \
  http://127.0.0.1/tsinspec/auth/login.do
핵심은 먼저 다음 주소가 정상인지 확인하는 것입니다.
http://127.0.0.1:9910/tsinspec/auth/login.do
이 주소가 정상인데 Apache 경유 접속만 실패하면 Apache ProxyPass 문제입니다. 이 주소부터 실패하면 Tomcat 기동·WAR 배포·DB 연결 문제입니다.


오전 2:16







Apache 를 8099 포트로 연결하려고 했는데 실패했는데?







오전 2:28






7초 동안 작업함






Apache 외부 포트를 8099, Tomcat 내부 포트를 9910으로 사용한다면 포트 역할을 다음처럼 분리해야 합니다.
사용자 → Apache :8099 → Tomcat 127.0.0.1:9910
Apache의 ProxyPass 대상에 8099를 넣으면 안 되고, Tomcat 포트인 9910을 넣어야 합니다.
1. 포트 사용 여부 확인
sudo ss -lntp \
  | grep -E ':8099|:9910'
정상 구성은 다음 형태입니다.
127.0.0.1:9910    java/Tomcat
0.0.0.0:8099      httpd
Apache를 시작하기 전 다른 프로세스가 8099를 사용하고 있다면 해당 프로세스와 충돌합니다.
2. Tomcat 직접 확인
Apache보다 먼저 Tomcat이 정상이어야 합니다.
curl -I \
  http://127.0.0.1:9910/tsinspec/auth/login.do
200 또는 302가 반환되어야 합니다. 연결 거부나 500이면 Apache 설정 전에 Tomcat 문제부터 해결해야 합니다.
3. Apache Listen 포트 설정
/etc/httpd/conf/httpd.conf를 확인합니다.
sudo grep -Rni \
  '^[[:space:]]*Listen' \
  /etc/httpd/conf \
  /etc/httpd/conf.d
8099 포트를 추가합니다.
Listen 8099
기존 Listen 80을 유지하면 Apache는 80과 8099를 모두 사용합니다.
Listen 80
Listen 8099
8099만 사용하려면 기존 Listen 80을 주석 처리할 수 있습니다.
#Listen 80
Listen 8099
동일한 Listen 8099가 여러 설정 파일에 중복으로 들어가면 안 됩니다.
4. VirtualHost 설정
/etc/httpd/conf.d/tsinspec.conf:
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
핵심 구분:
<VirtualHost *:8099>                         # Apache 수신 포트
ProxyPass ... http://127.0.0.1:9910/...      # Tomcat 포트
5. SELinux에 Apache 포트 등록
8099가 http_port_t에 등록되어 있는지 확인합니다.
sudo semanage port -l \
  | grep http_port_t
8099가 없으면 등록합니다.
sudo semanage port -a \
  -t http_port_t \
  -p tcp \
  8099
이미 다른 타입으로 등록되었다는 오류가 나오면 현재 등록을 확인합니다.
sudo semanage port -l \
  | grep -w 8099
다른 서비스가 사용하지 않는 포트임이 확인된 경우에만 변경합니다.
sudo semanage port -m \
  -t http_port_t \
  -p tcp \
  8099
Apache가 Tomcat으로 연결할 수 있도록 설정합니다.
sudo setsebool -P \
  httpd_can_network_connect \
  1
6. 방화벽 허용
8099 포트로 외부 접속해야 하므로 방화벽을 엽니다.
sudo firewall-cmd --permanent \
  --add-port=8099/tcp
sudo firewall-cmd --reload
확인:
sudo firewall-cmd \
  --query-port=8099/tcp
예상 결과:
yes
허용 IP만 접근시키려면 전체 포트 허용 대신 rich rule을 사용하는 것이 더 안전합니다.
sudo firewall-cmd --permanent \
  --remove-port=8099/tcp
sudo firewall-cmd --permanent \
  --add-rich-rule='rule family="ipv4" source address="<허용대역/CIDR>" port protocol="tcp" port="8099" accept'
sudo firewall-cmd --reload
7. Apache 설정 검사
sudo httpd -t
정상 결과:
Syntax OK
VirtualHost 인식 확인:
sudo httpd -S
모듈 확인:
sudo httpd -M \
  | grep -E 'proxy_module|proxy_http_module|headers_module'
8. Apache 재시작
sudo systemctl restart httpd
sudo systemctl status \
  httpd \
  --no-pager
포트 확인:
sudo ss -lntp \
  | grep ':8099'
9. 단계별 접속 확인
Apache 로컬 접속:
curl -I \
  http://127.0.0.1:8099/tsinspec/auth/login.do
루트 리다이렉트:
curl -I \
  http://127.0.0.1:8099/
외부 PC에서는 다음 주소를 사용합니다.
http://<서버IP>:8099/tsinspec/auth/login.do
오류별 원인
결과	주요 원인
Apache 시작 실패	Listen 중복, SELinux 포트 미등록, 포트 충돌
연결 거부	Apache가 8099에서 대기하지 않음 또는 방화벽
403	<Location>의 Require ip에 사용자 IP가 없음
404	ProxyPass 경로 또는 WAR 컨텍스트 오류
503	Tomcat 9910 미기동 또는 Apache의 Tomcat 연결 차단
200/302	정상


Apache 시작 실패 로그:
sudo journalctl \
  -u httpd \
  -n 200 \
  --no-pager
프록시 오류:
sudo tail -100 \
  /var/log/httpd/tsinspec_error.log
가장 먼저 다음 두 명령을 비교하면 됩니다.
curl -I http://127.0.0.1:9910/tsinspec/auth/login.do
curl -I http://127.0.0.1:8099/tsinspec/auth/login.do
첫 번째는 정상이고 두 번째만 실패하면 Apache의 Listen 8099, SELinux, 방화벽 또는 ProxyPass 설정 문제입니다.

이게 없으면?







오전 2:30






5초 동안 작업함






0.0.0.0:8099 httpd가 없으면 Apache가 8099 포트에서 대기하지 않는 상태입니다. <VirtualHost *:8099>만으로는 부족하고 반드시 Listen 8099가 있어야 합니다.
1. Apache 실행 상태 확인
sudo systemctl status httpd --no-pager
중지 상태라면 로그를 확인합니다.
sudo journalctl -u httpd -n 200 --no-pager
2. Listen 설정 확인
sudo grep -RniE \
  '^[[:space:]]*Listen[[:space:]]+' \
  /etc/httpd/conf \
  /etc/httpd/conf.d
다음 설정이 한 번만 있어야 합니다.
Listen 8099
/etc/httpd/conf/httpd.conf 또는 /etc/httpd/conf.d/tsinspec.conf에 추가합니다.
sudo vi /etc/httpd/conf.d/tsinspec.conf
예:
Listen 8099

<VirtualHost *:8099>
    ServerName <서버IP>

    ProxyRequests Off
    ProxyPreserveHost On

    ProxyPass        "/tsinspec/" "http://127.0.0.1:9910/tsinspec/"
    ProxyPassReverse "/tsinspec/" "http://127.0.0.1:9910/tsinspec/"
</VirtualHost>
설정 파일 이름이 tsinspec.conf.disabled라면 Apache가 읽지 않습니다. 반드시 .conf여야 합니다.
3. 설정 문법 검사
sudo httpd -t
정상 결과:
Syntax OK
VirtualHost 인식 확인:
sudo httpd -S
출력에 *:8099 VirtualHost가 표시되어야 합니다.
4. 포트 충돌 확인
sudo ss -lntp | grep ':8099'
다른 프로세스가 사용 중이면 Apache가 포트를 열 수 없습니다.
sudo lsof -nP -iTCP:8099 -sTCP:LISTEN
5. SELinux 포트 등록
sudo semanage port -l | grep -w 8099
결과가 없으면:
sudo semanage port -a \
  -t http_port_t \
  -p tcp \
  8099
이미 다른 SELinux 타입에 등록되어 있다면 해당 서비스 사용 여부를 확인한 후 변경합니다.
sudo semanage port -m \
  -t http_port_t \
  -p tcp \
  8099
6. Apache 재시작
sudo systemctl restart httpd
sudo systemctl status httpd --no-pager
다시 확인합니다.
sudo ss -lntp | grep ':8099'
정상적인 결과는 IPv4 또는 IPv6 형태 중 하나로 나올 수 있습니다.
0.0.0.0:8099
또는:
[::]:8099
[::]:8099만 보여도 시스템 설정에 따라 IPv4 연결을 함께 받을 수 있습니다.
7. 로컬 접속 확인
curl -I http://127.0.0.1:8099/
결과별 의미:
- 연결 거부: Apache가 8099에서 듣고 있지 않음
- 403: Apache는 정상이나 Require ip에서 차단
- 503: Apache는 정상이나 Tomcat 9910 연결 실패
- 200 또는 302: 정상
방화벽은 외부 접속에만 영향을 주므로 ss에 8099가 없는 문제와는 무관합니다. 우선 Listen 8099, httpd -t, SELinux 포트 등록, journalctl -u httpd 순으로 확인하면 됩니다.


오전 2:30







10월 15 01:00:59 localhost.localdomain systemd[1]: httpd.service: Deactivated successfully.
10월 15 01:00:59 localhost.localdomain systemd[1]: Stopped The Apache HTTP Server.
10월 15 01:00:59 localhost.localdomain systemd[1]: httpd.service: Consumed 1min 11.534s CPU time, 24.8M memory peak.
10월 15 02:26:20 localhost.localdomain systemd[1]: Starting The Apache HTTP Server...
10월 15 02:26:20 localhost.localdomain httpd[232946]: Server configured, listening on: port 443, port 80
10월 15 02:26:20 localhost.localdomain systemd[1]: Started The Apache HTTP Server.
10월 15 02:32:24 localhost.localdomain systemd[1]: Stopping The Apache HTTP Server...
10월 15 02:32:25 localhost.localdomain systemd[1]: httpd.service: Deactivated successfully.
10월 15 02:32:25 localhost.localdomain systemd[1]: Stopped The Apache HTTP Server.
10월 15 02:32:25 localhost.localdomain systemd[1]: Starting The Apache HTTP Server...
10월 15 02:32:25 localhost.localdomain httpd[233242]: (13)Permission denied: AH00072: make_sock: could not bind to address [::]:8099
10월 15 02:32:25 localhost.localdomain httpd[233242]: (13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:8099
10월 15 02:32:25 localhost.localdomain httpd[233242]: no listening sockets available, shutting down
10월 15 02:32:25 localhost.localdomain httpd[233242]: AH00015: Unable to open logs
10월 15 02:32:25 localhost.localdomain systemd[1]: httpd.service: Main process exited, code=exited, status=1/FAILURE
10월 15 02:32:25 localhost.localdomain systemd[1]: httpd.service: Failed with result 'exit-code'.
10월 15 02:32:25 localhost.localdomain systemd[1]: Failed to start The Apache HTTP Server.
이것은?







오전 2:33






3초 동안 작업함






원인은 SELinux가 Apache의 8099 포트 바인딩을 차단한 것입니다.
핵심 오류:
(13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:8099
Apache 설정 문법이나 파일 권한보다는 8099 포트가 http_port_t로 등록되지 않은 것이 원인입니다.
1. SELinux 상태 확인
getenforce
Enforcing이면 아래 설정이 필요합니다.
현재 포트 타입 확인:
sudo semanage port -l | grep -w 8099
2. 8099를 Apache 포트로 등록
아무 결과가 없다면:
sudo semanage port -a \
  -t http_port_t \
  -p tcp \
  8099
등록 확인:
sudo semanage port -l \
  | grep http_port_t \
  | grep -w 8099
3. 이미 다른 타입으로 등록된 경우
다음 명령에서 다른 SELinux 타입이 나온다면:
sudo semanage port -l | grep -w 8099
8099를 사용하는 다른 서비스가 없는지 확인합니다.
sudo ss -lntp | grep ':8099'
다른 서비스가 없고 Apache 전용으로 사용할 경우 변경합니다.
sudo semanage port -m \
  -t http_port_t \
  -p tcp \
  8099
4. semanage가 없는 경우
sudo dnf install policycoreutils-python-utils
설치 후 포트를 등록합니다.
5. Apache가 Tomcat 9910에 접속하도록 허용
8099 바인딩 허용과는 별개의 설정입니다.
sudo setsebool -P \
  httpd_can_network_connect \
  1
확인:
getsebool httpd_can_network_connect
예상 결과:
httpd_can_network_connect --> on
6. Apache 설정 검사 및 시작
sudo httpd -t
Syntax OK가 나오면:
sudo systemctl start httpd
sudo systemctl status httpd --no-pager
포트 확인:
sudo ss -lntp | grep ':8099'
정상 예:
0.0.0.0:8099
또는:
[::]:8099
7. 방화벽 설정
서버 외부에서 8099로 접속해야 한다면:
sudo firewall-cmd --permanent \
  --add-port=8099/tcp

sudo firewall-cmd --reload
확인:
sudo firewall-cmd \
  --query-port=8099/tcp
8. 최종 확인
Tomcat 직접 접속:
curl -I \
  http://127.0.0.1:9910/tsinspec/auth/login.do
Apache 경유:
curl -I \
  http://127.0.0.1:8099/tsinspec/auth/login.do
로그의 다음 메시지는 8099 바인딩 실패에 이어 발생한 부수적인 종료 메시지일 가능성이 큽니다.
AH00015: Unable to open logs
8099를 http_port_t로 등록한 후에도 AH00015가 계속 나오면 그때 /var/log/httpd 권한과 SELinux 컨텍스트를 별도로 확인하면 됩니다.
SELinux를 setenforce 0으로 끄거나 Apache를 임의로 root 권한 확대하여 해결하지 말고, 8099 포트를 http_port_t로 등록하는 것이 올바른 조치입니다.


