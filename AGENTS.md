# AGENTS — Running Online_Study_System locally

Online Study System is a Chinese-language online examination app (upstream:
https://github.com/jitwxs/Online_Study_System, Apache 2.0 — see `UPSTREAM.md`).
Stack: Spring MVC 4.3 + Shiro 1.2 + MyBatis-style DAO + JSP/JSTL, Java 8, Maven WAR.

This repo is a **full-history mirror** of upstream; tag `pristine` marks the
adopted upstream HEAD. All sample work lands via PR into `main`.

---

## 0. Prerequisites (absolute paths on this machine)

| Thing | Path / command | Notes |
|---|---|---|
| **JDK 8** (mandatory) | `/usr/lib/jvm/jdk-8.0.202-oracle-x64` | Never inherit the shell `JAVA_HOME` (machine default is JDK 21). |
| **Maven** | on `PATH` | |
| **MySQL 5.7** | docker `samples-mysql` (see below) | The dump targets MySQL 5.7; do not use MySQL 8. |
| **Tomcat 9** | `$TC_HOME` + shared samples base `$TC_BASE` | javax.servlet stack; Tomcat 10+ (jakarta) will NOT work. |

```bash
export JDK8=/usr/lib/jvm/jdk-8.0.202-oracle-x64
export TC_HOME=/home/ever/tools/apache-tomcat-9.0.83
export TC_BASE=/home/ever/tools/tomcat-samples-base
```

> `$TC_BASE` is shared by the three JSP samples (Online_Study_System,
> shopping-cart, Javaweb_bookstore). HTTP port **8081**, shutdown port **8006**.

---

## 1. Quick start (copy-paste)

```bash
# --- 1. MySQL 5.7 (root/root on localhost:3306) ------------------------------
docker start samples-mysql 2>/dev/null || docker run -d --name samples-mysql \
  -e MYSQL_ROOT_PASSWORD=root -p 3306:3306 mysql:5.7
sleep 20   # first boot initializes the datadir

# --- 2. Load the schema + seed (6 tables, utf8) ------------------------------
docker exec -i samples-mysql mysql -uroot -proot --default-character-set=utf8 \
  online_study_system < sql/online_study_system.sql
# (create the db first if the container is brand new:
#  docker exec samples-mysql mysql -uroot -proot \
#    -e "CREATE DATABASE IF NOT EXISTS online_study_system CHARACTER SET utf8")

# --- 3. Build the WAR (JDK 8) -------------------------------------------------
JAVA_HOME=$JDK8 mvn -B clean package -DskipTests   # -> target/Examination_System.war

# --- 4. Deploy + start Tomcat -------------------------------------------------
cp target/Examination_System.war $TC_BASE/webapps/
JAVA_HOME=$JDK8 CATALINA_HOME=$TC_HOME CATALINA_BASE=$TC_BASE \
  $TC_HOME/bin/catalina.sh start

# --- 5. Use it ---------------------------------------------------------------
# http://localhost:8081/Examination_System/   (redirects to login.jsp)
```

## 2. Verify

```bash
curl -so /dev/null -w '%{http_code} %{time_total}s\n' \
  http://localhost:8081/Examination_System/login.jsp   # 200 in < 1 s (warm)
```

Seeded logins (all passwords `123`, salted SHA-1 x1024 — see `SHA1Utils`):

| user | role |
|---|---|
| `admin` | 1 (admin) |
| `李老师`, `张老师` | 2 (teacher) |
| `小明` | 3 (student) |

DB config lives in `src/main/resources/mysql.properties`
(`root/root@localhost:3306/online_study_system`) — matches the container above.

## 3. Logs & teardown

```bash
tail -f $TC_BASE/logs/catalina.out
JAVA_HOME=$JDK8 CATALINA_HOME=$TC_HOME CATALINA_BASE=$TC_BASE $TC_HOME/bin/catalina.sh stop
```

## 4. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Data too long for column 'sex'` while loading the dump | Client charset was latin1 — load with `--default-character-set=utf8`. |
| 404 on every URL | Tomcat running on JDK 9+ — restart with `JAVA_HOME=$JDK8`. |
| Login always fails with 500 | MySQL down or wrong creds — check `mysql.properties` vs the container. |
| Port 8081 busy | Another sample Tomcat instance running — check `$TC_BASE` shutdown port 8006. |
