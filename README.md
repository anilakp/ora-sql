# Repository for oracle sql-related stuff

## Table of contents
1. [SQL various commands](#sql-various-commands)
2. [Configuring sqlplus sessions](#configuring-sqlplus-sessions)

### SQL various commands

* List available procedures for the given package
~~~
SELECT PROCEDURE_NAME, OWNER, OBJECT_NAME FROM DBA_PROCEDURES WHERE OBJECT_NAME=PACKAGE_NAME
~~~

* Check size of segments
~~~
SELECT S.OWNER "owner", NVL(S.SEGMENT_NAME, 'TABLE TOTAL SIZE') "segment_name",
    ROUND(SUM(S.BYTES)/1024/1024/1024,1) "size [GB]"
    FROM DBA_SEGMENTS S
    WHERE S.SEGMENT_NAME IN (SEGMENT_NAME)
        AND S.OWNER IN (SEGMENT_OWNER)
        OR S.SEGMENT_NAME IN (
            (
               SELECT L.SEGMENT_NAME FROM DBA_LOBS L WHERE L.TABLE_NAME = TABLE_NAME AND L.OWNER IN (OWNERS)
            )
    )
    GROUP BY S.OWNER, ROLLUP(S.SEGMENT_NAME)
    ORDER BY 1,2,3;
~~~

* Last activity time on a given table
~~~
SELECT MAX(ORA_ROWSCN), SCN_TO_TIMESTAMP(MAX(ORA_ROWSCN)) from OWNER_NAME.TABLE_NAME;
~~~

* Useful information about running sqls
~~~
SELECT * FROM (
    SELECT P.SPID "OSPID", (SE.SID),SS.SERIAL#,SS.SQL_ID,SS.USERNAME,SUBSTR(SS.PROGRAM,1,22) "PROGRAM",
    SS.MODULE,SS.OSUSER,SS.MACHINE,SS.STATUS, SE.VALUE/100 CPU_USAGE_SEC
    FROM
        V$SESSION SS,
        V$SESSTAT SE,
        V$STATNAME SN,
        V$PROCESS P
    WHERE
        SE.STATISTIC# = SN.STATISTIC#
        AND
        NAME LIKE '%cpu USED BY THIS SESSION%'
        AND
        SE.SID = SS.SID
        AND SS.USERNAME !='sys' AND
        SS.STATUS='active'
        AND SS.USERNAME IS NOT NULL
        AND SS.PADDR=P.ADDR AND VALUE > 0
    ORDER BY SE.VALUE DESC);
~~~

* Number of rows per partition of a table
~~~
SELECT table_owner, partition_name, num_rows, last_analyzed from all_tab_partitions WHERE
table_owner IN (...) AND table_name=TABLE_NAME ORDER BY table_owner, partition_name DESC;

~~~

* Check size of temporary tablespace
~~~
SELECT tablespace_name,
    round((tablespace_size)/1024/1024 ,2) AS "total size [MB]",
    round((allocated_space)/1024/1024 ,2) AS "allocated [MB]",
    round((free_space)/1024/1024 ,2) as "free space [MB]"
FROM dba_temp_free_space;
~~~

* Add datafile to the tablespace to be able to extend it
~~~
ALTER TABLESPACE TABLESPACE_NAME ADD DATAFILE '/data/oracle/db18c/oradata/NEW_FILE_NAME.dbf' SIZE 10g;
~~~

* Check datafiles per tablespace
~~~
SELECT NAME, FILE#, STATUS, CHECKPOINT_CHANGE# "CHECKPOINT"
    FROM  V$DATAFILE ORDER BY name;
~~~

* Tables for checking privileges grant
~~~
all_tab_privs, dba_sys_privs, dba_role_privs
~~~

* Investigation of MVs issues
~~~
SELECT * FROM DBA_JOBS_RUNNING;

SELECT JOB, LAST_DATE, NEXT_DATE, INTERVAL, FAILURES, WHAT FROM DBA_JOBS WHERE SCHEMA_USER =
OWNER_NAME AND WHAT LIKE 'dbms_refresh.refresh(%';

SELECT JOB, WHAT FROM DBA_JOBS WHERE BROKEN='Y'

select dbr.JOB,s.username, dbr.FAILURES, dbj.BROKEN from dba_jobs_running dbr, dba_jobs dbj,
v$session s where dbr.job=dbj.job and s.sid=dbr.sid;
~~~

* Convert partition high value to 'date' type
~~~
-- Function returns high_value of a given partition as date
-- In case partitioning is done against non-date column 
-- following exception is thrown:
-- ORA-01858: a non-numeric character was found where a numeric was expected
-- NOTE: Exception handling is supposed to be done at application level
CREATE OR REPLACE FUNCTION PART_HV_TO_DATE (p_table_owner IN VARCHAR2,
 p_table_name IN VARCHAR2,
 p_partition_name IN VARCHAR2)
 RETURN DATE
AS
 l_high_value VARCHAR2(32767);
 l_date DATE;
BEGIN
 SELECT high_value
 INTO l_high_value
 FROM all_tab_partitions
 WHERE table_owner = p_table_owner
 AND table_name = p_table_name
 AND partition_name = p_partition_name;

 EXECUTE IMMEDIATE 'SELECT ' || l_high_value || ' FROM dual' INTO l_date;
 RETURN l_date;
END;
/
~~~


### Configuring sqlplus sessions

1. Place it in your home directory
2. To enable login.sql for every sqlplus session, I did the following alias:
~~~
alias sqlcon='cd ~ && rlwrap sqlplus user/pwd@host:port/instance; cd -'
~~~
NOTE: [Rlwrap](https://github.com/hanslub42/rlwrap) is a readline wrapper, useful when combining
with sqlplus cli, enabling you with many possibilities - must have for me, which is navigation through arrows: over the history
