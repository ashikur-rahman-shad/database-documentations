Prerequisites: Make sure oracle database is installed. This notebook was created for Oracle 19c.

### Open oracle SQL shell
Switch to oracle user
```bash
su - oracle
```

Open sqlplus as `SYS`:
```bash
sqlplus / as sysdba
```

### Enable  ARCHIVELOG

Check if ARCHIVELOG is enabled:
```sql
ALTER SESSION SET CONTAINER = CDB$ROOT;
SELECT log_mode FROM v$database;
```
If not enabled, enable it
```sql
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
```
### Enable Supplemental Logging

check if supplemental logging is enabled
```sql
ALTER SESSION SET CONTAINER = CDB$ROOT;
SELECT supplemental_log_data_min, supplemental_log_data_all FROM v$database;
```
If not enable, enable it
```sql
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
```

### Enable GoldenGate Replication (Required for LogMiner)

Check if already enabled
```sql
ALTER SESSION SET CONTAINER = CDB$ROOT;
SHOW PARAMETER enable_goldengate_replication;
```
If not enabled, enable it:
```sql
ALTER SYSTEM SET enable_goldengate_replication = TRUE SCOPE=BOTH;
```

### Create User Debezium

```sql
ALTER SESSION SET CONTAINER = CDB$ROOT;
SELECT username FROM dba_users WHERE username = 'C##DEBEZIUM';
```
If doesn't exists, create one
```sql
CREATE USER C##DEBEZIUM IDENTIFIED BY Debezium2026# CONTAINER=ALL;
```
*Here, C##DEBEZIUM is the username and Debezium2026# is the password*

Grant Privileges
```sql
ALTER SESSION SET CONTAINER = CDB$ROOT;
GRANT SET CONTAINER TO C##DEBEZIUM CONTAINER=ALL;
GRANT CREATE SESSION TO C##DEBEZIUM CONTAINER=ALL;
GRANT SELECT ANY TABLE TO C##DEBEZIUM CONTAINER=ALL;
GRANT SELECT ANY TRANSACTION TO C##DEBEZIUM CONTAINER=ALL;
GRANT FLASHBACK ANY TABLE TO C##DEBEZIUM CONTAINER=ALL;
GRANT LOGMINING TO C##DEBEZIUM CONTAINER=ALL;
GRANT EXECUTE_CATALOG_ROLE TO C##DEBEZIUM CONTAINER=ALL;
GRANT CREATE TABLE TO C##DEBEZIUM CONTAINER=ALL;
ALTER USER C##DEBEZIUM QUOTA UNLIMITED ON USERS;
GRANT SELECT ANY DICTIONARY TO C##DEBEZIUM CONTAINER=ALL;
GRANT EXECUTE ON SYS.DBMS_LOGMNR   TO C##DEBEZIUM CONTAINER=ALL;
GRANT EXECUTE ON SYS.DBMS_LOGMNR_D TO C##DEBEZIUM CONTAINER=ALL;
GRANT SELECT ON SYS.V_$DATABASE          TO C##DEBEZIUM CONTAINER=ALL;
GRANT SELECT ON SYS.V_$LOG               TO C##DEBEZIUM CONTAINER=ALL;
GRANT SELECT ON SYS.V_$LOGFILE           TO C##DEBEZIUM CONTAINER=ALL;
GRANT SELECT ON SYS.V_$ARCHIVED_LOG      TO C##DEBEZIUM CONTAINER=ALL;
GRANT SELECT ON SYS.V_$TRANSACTION       TO C##DEBEZIUM CONTAINER=ALL;
GRANT SELECT ON SYS.V_$MYSTAT            TO C##DEBEZIUM CONTAINER=ALL;
GRANT SELECT ON SYS.V_$STATNAME          TO C##DEBEZIUM CONTAINER=ALL;
GRANT SELECT ON SYS.V_$INSTANCE          TO C##DEBEZIUM CONTAINER=ALL;
GRANT SELECT ON SYS.V_$VERSION           TO C##DEBEZIUM CONTAINER=ALL;
GRANT SELECT ON SYS.V_$PARAMETER         TO C##DEBEZIUM CONTAINER=ALL;
GRANT SELECT ON SYS.V_$ARCHIVE_DEST_STATUS TO C##DEBEZIUM CONTAINER=ALL;

ALTER SESSION SET CONTAINER = ORCLPDB1;
GRANT SELECT_CATALOG_ROLE TO C##DEBEZIUM;

```

### Enable supplementary logging for user tables

 Example:
```sql
ALTER SESSION SET CONTAINER = ORCLPDB1;
ALTER TABLE HR.EMPLOYEES ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
GRANT SELECT ON DE.TRANSACTIONS TO C##DEBEZIUM;
```
Here HR is the user name and EMPLOYEES is the table name.

It's highly recommended to have a primary key in your table. Debezium works best when you have a primary key. If you don't have one, add it.

### Test Debezium access to database

Try
```sql
CONNECT C##DEBEZIUM/Debezium2026#@ORCLPDB1
```
if the above doesn't work, try the following:
```sql
CONNECT C##DEBEZIUM/Debezium2026#@//localhost:1521/ORCLPDB1
```

Check if Debezium can read your tables or not:
```sql
SELECT COUNT(*) FROM DE.TRANSACTIONS;
```

Check if Debezium can access LogMiner:
```sql
SELECT COUNT(*) FROM v$archived_log;
```
