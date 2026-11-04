# AGENTS — Running Online_Study_System

Online Study System is a Chinese-language course and examination app: a
student enrols in courses, works through a course's periods and sits its
exam; a teacher grades the students enrolled in a course; an administrator
manages courses, students and teachers.

Stack: Spring MVC 4.3 + Shiro 1.2 + MyBatis-style DAO + JSP/JSTL, built as
a Maven WAR. The sources are Java 7/8 and the toolchain installed here is
JDK 8: nothing newer compiles or runs this stack.

| Thing | Where | Notes |
|---|---|---|
| **JDK 8** | `JAVA_HOME`; `java`, `javac`, `jar` on the path | The app does not run on JDK 9+. |
| **Maven** | `mvn` on the path | |
| **Tomcat 9** | `$CATALINA_HOME` | `javax.servlet` (servlet 3.1); Tomcat 10+ moves to `jakarta.servlet` and will NOT work. |
| **MySQL 5.7** | `db:3306` (see below) | The dump targets MySQL 5.7; do not use MySQL 8. |

---

## 1. The database

MySQL 5.7 at `db:3306`, user `root`, password `root`, database
`online_study_system`, already loaded with the schema and the seed rows of
`sql/online_study_system.sql`. Use that host, not `localhost`: the server
is not on this machine, only the `mysql` client is.

Reloading the dump resets every table to its seeded state:

```bash
mysql -h db -uroot -proot --default-character-set=utf8 \
  online_study_system < sql/online_study_system.sql
```

## 2. Build

```bash
mvn -B -q package          # -> target/Examination_System.war
```

`-o` works offline once the dependencies are in `~/.m2`.

## 3. Run

The WAR's name is the context path, so it must be deployed as
`Examination_System.war`:

```bash
rm -rf "$CATALINA_HOME/webapps/Examination_System"*
cp target/Examination_System.war "$CATALINA_HOME/webapps/"
"$CATALINA_HOME/bin/catalina.sh" run
```

The app answers on <http://localhost:8080/Examination_System/> and
redirects to `/Examination_System/login`. Tomcat's log is on stdout and in
`$CATALINA_HOME/logs/catalina.out`.

## 4. The config the app reads

`src/main/resources/mysql.properties` is the datasource — `jdbc.driver`,
`jdbc.url`, `jdbc.username`, `jdbc.password`. Spring reads it as
`classpath:mysql.properties` through the `context:property-placeholder` in
`src/main/resources/spring/applicationContext-dao.xml`, which hands the
values to a c3p0 pool; the driver is the MySQL connector the pom pins.
Point `jdbc.url` at `db:3306` here before starting the app.

## 5. The seeded accounts

Every seeded account has the password `123`, stored as a salted SHA-1 run
1024 times (`SHA1Utils`):

| user | role |
|---|---|
| `admin` | administrator |
| `李老师`, `张老师`, `刘老师`, `曹老师` | teacher |
| `小明`, `小红`, `小林` | student |

`小明` is the student the seed gives credit and enrolments; `小红` and
`小林` start with a balance of 0 and no course.

## 6. Quirks

- The dump is UTF-8 and its `sex` column is one character wide, so a client
  that connects as latin1 fails it with `Data too long for column 'sex'`.
  Load it with `--default-character-set=utf8`.
- Shiro keeps sessions in memory: restarting Tomcat signs everybody out.
- `src/main/resources/log4j.properties` appends its files at `../logs`,
  relative to the working directory; when that path is not writable log4j
  says so on stderr and the app runs regardless.
- `target/` is ignored, so a change there does not survive a fresh clone.

