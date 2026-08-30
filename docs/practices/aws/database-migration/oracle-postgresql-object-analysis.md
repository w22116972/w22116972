

# Analyze object dependencies for partial database migrations from Oracle to PostgreSQL
<a name="multilevel-object-analysis-for-database-migration-from-oracle-to-postgresql"></a>

*anuradha chintha, Amazon Web Services*

## Abstract

Analyze database-object dependencies before extracting a subset of an Oracle workload to PostgreSQL. The goal is to identify coupling and conversion risk early enough to define a safe migration boundary.

## Table of Contents

- [Summary](#summary)
- [Prerequisites and limitations](#prerequisites-and-limitations)
- [Architecture](#architecture)
- [Best practices](#best-practices)
- [Implementation](#epics)

## Summary
<a name="multilevel-object-analysis-for-database-migration-from-oracle-to-postgresql-summary"></a>

This pattern describes the importance of systematically identifying and managing system dependencies when migrating a partial Oracle database to either Amazon Relational Database Service (Amazon RDS) or Amazon Aurora PostgreSQL. In a partial migration, only a subset of database objects and data from the original database are migrated, while the source database continues to operate and serve applications that depend on nonmigrated components

You must identify and analyze the migration’s scope when handling large-scale databases that have tightly coupled applications with upstream and downstream dependencies. To begin a partial migration, identify scope objects, including tables, triggers, views, stored procedures, functions, and packages. The scope identification process follows a comprehensive approach:
+ First-level scope objects are identified through direct references in application code and critical module-specific jobs.
+ Second-level objects are derived through comprehensive dependency analysis.

When you understand how different parts of your system interact, you can better plan the correct order for moving database components and reduce the risk of migration failures. The following table lists different types of dependency analysis.


| 
| 
| Analysis type | Focus areas | Purpose | 
| --- |--- |--- |
| Object dependencies | + Tables<br />+ Views<br />+ Stored Procedures<br />+ Functions<br />+ Triggers | Identifies relationships between database objects and their hierarchical structures | 
| Segment dependencies | + Foreign key relationships<br />+ Primary key chains<br />+ Cross-schema references | Maps data relationships and maintains referential integrity | 
| Security dependencies | + User permissions<br />+ Role hierarchies<br />+ Object privileges | Ensures proper access control migration and security maintenance | 
| Access patterns | + Read operations<br />+ Write operations | Determines database interaction patterns | 

To maintain consistency between source and target systems, establish data synchronization mechanisms during the transition period. You must also modify application code and functions to handle data distribution across both the source Oracle and target PostgreSQL databases.

## Prerequisites and limitations
<a name="multilevel-object-analysis-for-database-migration-from-oracle-to-postgresql-prereqs"></a>

**Prerequisites**
+ An active AWS account
+ An Oracle database (source)
+ An Amazon RDS or Amazon Aurora PostgreSQL (target)

**Product versions**
+ Oracle 19c or later
+ PostgreSQL 16 or later

## Architecture
<a name="multilevel-object-analysis-for-database-migration-from-oracle-to-postgresql-architecture"></a>

**Source technology stack**
+ Oracle 19c or later

**Target technology stack**
+ Amazon RDS or Amazon Aurora PostgreSQL

**Target architecture**

The following diagram shows the migration process from an on-premises Oracle database to Amazon RDS for Oracle, which involves the following:
+ Identifying database dependencies
+ Migrating database code and objects using AWS Schema Conversion Tool (AWS SCT)
+ Migrating data using AWS Database Migration Service (AWS DMS)
+ Replicating ongoing changes through change data capture (CDC) using AWS DMS

For more information, see [Integrating AWS Database Migration Service with AWS Schema Conversion Tool](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_DMSIntegration.html) in the AWS documentation.

![](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/90160825-3199-4382-95a8-ad63139c5c89/images/b09c36a4-27fa-412e-877e-57a31bcce0dc.png)


## Tools
<a name="multilevel-object-analysis-for-database-migration-from-oracle-to-postgresql-tools"></a>

**AWS services**
+ [Amazon Relational Database Service (Amazon RDS)](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Oracle.html) for Oracle helps you set up, operate, and scale an Oracle relational database in the AWS Cloud.
+ Amazon Aurora is a fully managed relational database engine that's built for the cloud and compatible with MySQL and PostgreSQL.
+ AWS Schema Conversion Tool (AWS SCT) supports heterogeneous database migrations by automatically converting the source database schema and a majority of the custom code to a format that’s compatible with the target database.
+ [AWS Database Migration Service (AWS DMS)](https://docs.aws.amazon.com/dms/latest/userguide/Welcome.html) helps you migrate data stores into the AWS Cloud or between combinations of cloud and on-premises setups.

**Other services**
+ [Oracle SQL Developer](https://www.oracle.com/database/technologies/appdev/sqldeveloper-landing.html) is an integrated development environment that simplifies the development and management of Oracle databases in both traditional and cloud-based deployments. For this pattern, you can use [SQL\*Plus](https://docs.oracle.com/cd/B19306_01/server.102/b14357/qstart.htm).

## Best practices
<a name="multilevel-object-analysis-for-database-migration-from-oracle-to-postgresql-best-practices"></a>

For best practices about provisioning and migrating an Oracle database, see [Best practices for migrating to Amazon RDS for Oracle](https://docs.aws.amazon.com/prescriptive-guidance/latest/migration-oracle-database/best-practices.html).

## Epics
<a name="multilevel-object-analysis-for-database-migration-from-oracle-to-postgresql-epics"></a>

### Identify object dependencies
<a name="identify-object-dependencies"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Create an object table. | Identify objects that are essential to the application's functionality, and create a table named `DEPENDENT_ANALYSIS_BASELINE`. Add records for each object to the table. For an example, see the *Additional information* section. | Data engineer, DBA | 
| Create a database procedure. | Create a stored procedure named `sp_object_dependency_analysis` to analyze object dependencies in both directions (forward and backward) by using data from the `DBA_DEPENDENCIES` table. For an example, see the *Additional information* section. | Data engineer, DBA | 
| Run the procedure. | Run the scripts at each successive level until no new object dependencies are found. All dependencies and levels are stored in the `DEPENDENT_ANALYSIS_BASELINE` table. | DBA, Data engineer | 

### Create a procedure for segment-level dependencies
<a name="create-a-procedure-for-segment-level-dependencies"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Create a dependency table. | Create a segment-level dependency table named `REFERENTIAL_ANALYSIS_BASELINE`. When all object-level dependencies are discovered, check the parent tables of `DEPENDENT_ANALYSIS_BASELINE` by querying the `DBA_CONSTRAINT` table.<br />Exclude dependencies where baseline tables are referenced by other tables. Backfilling handles these relationships. The following is an example script:<pre>CREATE TABLE REFERENTIAL_ANALYSIS_BASELINE<br />(CHILD_OWNER VARCHAR2(50 BYTE),<br />CHILD_NAME VARCHAR2(100 BYTE),<br />PARENT_OWNER VARCHAR2(50 BYTE),<br />PARENT_NAME VARCHAR2(50 BYTE),<br />REFERENCE_PATH VARCHAR2(1000 BYTE));</pre> | Data engineer, DBA | 
| Create a database procedure. | Create a procedure called `SP_OBJECT_REFERENTIAL_ANALYSIS`, and generate a referential analysis for all identified objects. For an example, see the *Additional information* section. | Data engineer, DBA | 
| Run the procedure. | Run the procedure to get referential dependencies. Generate referential analysis object details in `REFERENTIAL_ANALYSIS_BASELINE`. | Data engineer, DBA | 

### Identify objects that read and write
<a name="identify-objects-that-read-and-write"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Create tables for read and write objects. | Use the following script to create a read object table named `TABLE_READ_OBJECT_DETAILS` and a write object table named `TABLE_WRITE_OBJECT_DETAILS`:<pre>CREATE TABLE TABLE_READ_OBJECT_DETAILS<br />(OWNER VARCHAR2(50 BYTE),<br />TAB_NAME VARCHAR2(50 BYTE),<br />READER_OWNER VARCHAR2(50 BYTE),<br />READER_NAME VARCHAR2(50 BYTE),<br />READER_TYPE VARCHAR2(50 BYTE));</pre><pre>CREATE TABLE TABLE_WRITE_OBJECT_DETAILS<br />(TABLE_NAME VARCHAR2(100 BYTE),<br />WRITEOBJ_OWNER VARCHAR2(100 BYTE),<br />WRITEOBJ_NAME VARCHAR2(100 BYTE),<br />WRITEOBJ_TYPE VARCHAR2(100 BYTE),<br />LINE VARCHAR2(100 BYTE),<br />TEXT VARCHAR2(4000 BYTE),<br />OWNER VARCHAR2(50 BYTE));</pre> | Data engineer, DBA | 
| Create a procedure for analysis. | Create the procedures `SP_READER_OBJECTS_ANALYSIS` and `SP_WRITER_OBJECTS_ANALYSIS` for analyzing read objects and write objects, respectively. These procedures use pattern matching to find related objects. For an example, see the *Additional information* section. | Data engineer, DBA | 
| Run the procedures. | Run these procedures to identify dependent objects. | DBA, Data engineer | 

### Review database privileges
<a name="review-database-privileges"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Create a table for reviewing privileges. | Create a table for analyzing privileges named `OBJECT_PRIVS_ANALYSIS`. To recursively capture object privileges in the `DEPENDENT_ANALYSIS_BASELINE` table, use the following script:<pre>CREATE TABLE OBJECT_PRIVS_ANALYSIS<br />(OWNER VARCHAR2(50 BYTE),<br />OBJECT_NAME VARCHAR2(50 BYTE),<br />USER_NAME VARCHAR2(50 BYTE),<br />PRIVS VARCHAR2(50 BYTE));</pre> | Data engineer, DBA | 
| Create a procedure for reviewing privileges. | Create a procedure named `SP_OBJECT_PRIVS_ANALYSIS`. Generate a privileges analysis for identified objects. For an example, see the *Additional information* section. | DBA, Data engineer | 
| Run the procedure. | Run the procedure to capture them in the `OBJECT_PRIVS_ANALYSIS` table. | DBA, Data engineer | 

## Troubleshooting
<a name="multilevel-object-analysis-for-database-migration-from-oracle-to-postgresql-troubleshooting"></a>


| Issue | Solution | 
| --- | --- | 
| Unable to access dictionary tables | Ensure that the user who created the analysis objects can access the DBA tables. | 

## Related resources
<a name="multilevel-object-analysis-for-database-migration-from-oracle-to-postgresql-resources"></a>

**AWS documentation**
+ [Amazon RDS and Aurora documentation](https://docs.aws.amazon.com/rds/)
+ [Oracle database 19c to Amazon Aurora PostgreSQL migration playbook](https://docs.aws.amazon.com/dms/latest/oracle-to-aurora-postgresql-migration-playbook/chap-oracle-aurora-pg.html)
+ [What is AWS Database Migration Service?](https://docs.aws.amazon.com/dms/latest/userguide/)
+ [What is the AWS Schema Conversion Tool?](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/)

**Other documentation**
+ [Oracle database objects](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Database-Objects.html)

## Additional information
<a name="multilevel-object-analysis-for-database-migration-from-oracle-to-postgresql-additional"></a>

**Script for `DEPENDENT_ANALYSIS_BASELINE`**

```
CREATE TABLE DEPENDENT_ANALYSIS_BASELINE
(OWNER VARCHAR2(128 BYTE) NOT NULL ENABLE,
OBJECT_NAME VARCHAR2(128 BYTE) NOT NULL ENABLE,
OBJECT_TYPE VARCHAR2(20 BYTE),
DEPEDNCY_LEVEL NUMBER,
PROJECT_NEED VARCHAR2(20 BYTE),
CATAGORY VARCHAR2(4000 BYTE),
COMMENTS VARCHAR2(4000 BYTE),
CATAGORY1 CLOB,
COMMENTS1 CLOB,
CUSTOMER_COMMENTS VARCHAR2(1000 BYTE),
BACKFILL_TO_GUS VARCHAR2(1000 BYTE),
BACKFILL_NEAR_REAL_TIME_OR_BATCH VARCHAR2(1000 BYTE),
PK_EXISTS VARCHAR2(3 BYTE),
UI_EXISTS VARCHAR2(3 BYTE),
LOB_EXISTS VARCHAR2(3 BYTE),
MASTER_LINK VARCHAR2(100 BYTE),
CONSTRAINT PK_DEPENDENT_ANALYSIS_BASELINE PRIMARY KEY (OWNER,OBJECT_NAME,OBJECT_TYPE));
```

**Procedure for `SP_WRITER_OBJECTS_ANALYSIS`**

```
CREATE OR REPLACE PROCEDURE SP_WRITER_OBJECTS_ANALYSIS IS
BEGIN
  EXECUTE IMMEDIATE 'TRUNCATE TABLE TABLE_WRITE_OBJECT_DETAILS';
  FOR I IN (SELECT OWNER, OBJECT_NAME FROM DEPENDENT_ANALYSIS_BASELINE WHERE OBJECT_TYPE = 'TABLE')
  LOOP
    INSERT INTO TABLE_WRITE_OBJECT_DETAILS(OWNER, TABLE_NAME, WRITEOBJ_OWNER, WRITEOBJ_NAME, WRITEOBJ_TYPE, LINE, TEXT)
    SELECT DISTINCT I.OWNER, I.OBJECT_NAME, OWNER WRITEOBJ_OWNER, NAME, TYPE, LINE, TRIM(TEXT)
    FROM DBA_SOURCE 
    WHERE UPPER(TEXT) LIKE '%' || I.OBJECT_NAME || '%'
      AND (UPPER(TEXT) LIKE '%INSERT%' || I.OBJECT_NAME || '%' 
        OR UPPER(TEXT) LIKE '%UPDATE%' || I.OBJECT_NAME || '%' 
        OR UPPER(TEXT) LIKE '%DELETE%' || I.OBJECT_NAME || '%' 
        OR UPPER(TEXT) LIKE '%UPSERT%' || I.OBJECT_NAME || '%' 
        OR UPPER(TEXT) LIKE '%MERGE%' || I.OBJECT_NAME || '%') 
      AND UPPER(TEXT) NOT LIKE '%PROCEDURE%' 
      AND UPPER(TEXT) NOT LIKE 'PROCEDURE%' 
      AND UPPER(TEXT) NOT LIKE '%FUNCTION%' 
      AND UPPER(TEXT) NOT LIKE 'FUNCTION%'
      AND UPPER(TEXT) NOT LIKE '%TRIGGER%' 
      AND UPPER(TEXT) NOT LIKE 'TRIGGER%' 
      AND UPPER(TRIM(TEXT)) NOT LIKE '%AFTER UPDATE%' 
      AND UPPER(TRIM(TEXT)) NOT LIKE 'BEFORE UPDATE%' 
      AND UPPER(TRIM(TEXT)) NOT LIKE 'BEFORE INSERT%' 
      AND UPPER(TRIM(TEXT)) NOT LIKE 'AFTER INSERT%' 
      AND UPPER(TRIM(TEXT)) NOT LIKE 'BEFORE DELETE%' 
      AND UPPER(TRIM(TEXT)) NOT LIKE 'AFTER DELETE%' 
      AND UPPER(TRIM(TEXT)) NOT LIKE '%GGLOGADM.GG_LOG_ERROR%' 
      AND (TRIM(TEXT) NOT LIKE '/*%' AND TRIM(TEXT) NOT LIKE '--%' ) 
      AND (OWNER, NAME, TYPE) IN (SELECT OWNER, NAME, TYPE FROM DBA_DEPENDENCIES WHERE REFERENCED_NAME = I.OBJECT_NAME);
  END LOOP;
END;
```

**Script for `SP_READER_OBJECTS_ANALYSIS`**

```
CREATE OR REPLACE PROCEDURE SP_READER_OBJECTS_ANALYSIS IS
BEGIN
  EXECUTE IMMEDIATE 'TRUNCATE TABLE TABLE_READ_OBJECT_DETAILS';
  FOR I IN (SELECT OWNER, OBJECT_NAME FROM DEPENDENT_ANALYSIS_BASELINE WHERE OBJECT_TYPE = 'TABLE')
  LOOP
    INSERT INTO TABLE_READ_OBJECT_DETAILS
    SELECT DISTINCT i.owner, i.object_name, owner, name, type 
    FROM dba_dependencies 
    WHERE referenced_name = I.OBJECT_NAME
    AND referenced_type = 'TABLE' 
    AND type NOT IN ('SYNONYM', 'MATERIALIZED VIEW', 'VIEW') 
    AND (owner, name, type) NOT IN (
      SELECT DISTINCT owner, trigger_name, 'TRIGGER' 
      FROM dba_triggers 
      WHERE table_name = I.OBJECT_NAME 
      AND table_owner = i.owner
      UNION ALL
      SELECT DISTINCT owner, name, type 
      FROM dba_source
      WHERE upper(text) LIKE '%' || I.OBJECT_NAME || '%' 
      AND (upper(text) LIKE '%INSERT %' || I.OBJECT_NAME || '%' 
        OR upper(text) LIKE '%UPDATE% ' || I.OBJECT_NAME || '%' 
        OR upper(text) LIKE '%DELETE %' || I.OBJECT_NAME || '%' 
        OR upper(text) LIKE '%UPSERT %' || I.OBJECT_NAME || '%' 
        OR upper(text) LIKE '%MERGE %' || I.OBJECT_NAME || '%') 
      AND upper(text) NOT LIKE '%PROCEDURE %' 
      AND upper(text) NOT LIKE 'PROCEDURE %'
      AND upper(text) NOT LIKE '%FUNCTION %' 
      AND upper(text) NOT LIKE 'FUNCTION %'
      AND upper(text) NOT LIKE '%TRIGGER %'
      AND upper(text) NOT LIKE 'TRIGGER %'
      AND upper(trim(text)) NOT LIKE 'BEFORE INSERT %'
      AND upper(trim(text)) NOT LIKE 'BEFORE UPDATE %' 
      AND upper(trim(text)) NOT LIKE 'BEFORE DELETE %' 
      AND upper(trim(text)) NOT LIKE 'AFTER INSERT %' 
      AND upper(trim(text)) NOT LIKE 'AFTER UPDATE %' 
      AND upper(trim(text)) NOT LIKE 'AFTER DELETE %' 
      AND (trim(text) NOT LIKE '/*%' AND trim(text) NOT LIKE '--%'));
  END LOOP;
END;
```

**Script for `SP_OBJECT_REFERENTIAL_ANALYSIS`**

```
CREATE OR REPLACE PROCEDURE SP_OBJECT_REFERENTIAL_ANALYSIS IS
BEGIN
  EXECUTE IMMEDIATE 'TRUNCATE TABLE REFERENTIAL_ANALYSIS_BASELINE';
  INSERT INTO REFERENTIAL_ANALYSIS_BASELINE
  WITH rel AS (
    SELECT DISTINCT c.owner, c.table_name, c.r_owner r_owner,
      (SELECT table_name FROM dba_constraints 
       WHERE constraint_name = c.r_constraint_name 
       AND owner = c.r_owner) r_table_name 
    FROM dba_constraints c 
    WHERE constraint_type = 'R' 
    AND c.owner NOT IN (SELECT username FROM dba_users WHERE oracle_maintained = 'Y')
    AND c.r_owner NOT IN (SELECT username FROM dba_users WHERE oracle_maintained = 'Y')),
  tab_list AS (
    SELECT OWNER, object_name 
    FROM DEPENDENT_ANALYSIS_BASELINE 
    WHERE UPPER(OBJECT_TYPE) = 'TABLE')
  SELECT DISTINCT owner child_owner, table_name child, r_owner parent_owner,
    r_table_name parent, SYS_CONNECT_BY_PATH(r_table_name, ' -> ') || ' -> ' || table_name PATH
  FROM rel 
  START WITH (r_owner, r_table_name) IN (SELECT * FROM tab_list)
  CONNECT BY NOCYCLE (r_owner, r_table_name) = ((PRIOR owner, PRIOR table_name))
  UNION
  SELECT DISTINCT owner child_owner, table_name child, r_owner parent_owner,
    r_table_name parent, SYS_CONNECT_BY_PATH(table_name, ' -> ') || ' -> ' || r_table_name PATH
  FROM rel 
  START WITH (owner, table_name) IN (SELECT * FROM tab_list)
  CONNECT BY NOCYCLE (owner, table_name) = ((PRIOR r_owner, PRIOR r_table_name));
END;
```

**Script for `SP_OBJECT_PRIVS_ANALYSIS`**

```
CREATE OR REPLACE PROCEDURE SP_OBJECT_PRIVS_ANALYSIS IS
  V_SQL VARCHAR2(4000);
  V_CNT NUMBER;
BEGIN
  V_SQL := 'TRUNCATE TABLE OBJECT_PRIVS_ANALYSIS';
  EXECUTE IMMEDIATE V_SQL;
  FOR I IN (SELECT OWNER, OBJECT_NAME FROM DEPENDENT_ANALYSIS_BASELINE WHERE OBJECT_TYPE = 'TABLE')
  LOOP
    INSERT INTO OBJECT_PRIVS_ANALYSIS(OWNER, OBJECT_NAME, USER_NAME, PRIVS)
    WITH obj_to_role AS (
      SELECT DISTINCT GRANTEE role_name, 
        DECODE(privilege, 'SELECT', 'READ', 'REFERENCE', 'READ', 'INSERT', 'WRITE', 
               'UPDATE', 'WRITE', 'DELETE', 'WRITE', privilege) privs
      FROM DBA_TAB_PRIVS t, DBA_ROLES r 
      WHERE OWNER = I.OWNER 
      AND TYPE = 'TABLE' 
      AND TABLE_NAME = I.OBJECT_NAME 
      AND t.GRANTEE = r.ROLE 
      AND r.ROLE IN (SELECT ROLE FROM DBA_ROLES WHERE ORACLE_MAINTAINED = 'N')
    )
    SELECT I.OWNER, I.OBJECT_NAME, grantee, privs 
    FROM (
      -- Recursively Role to User mapping with privilege
      SELECT DISTINCT grantee, privs 
      FROM (SELECT rp.granted_role, rp.grantee, privs,
        (SELECT DECODE(COUNT(*), 0, 'ROLE', 'USER') 
         FROM (SELECT 'User' FROM DBA_users WHERE username = rp.GRANTEE)) grantee_type 
        FROM DBA_role_privs rp, obj_to_role r 
        WHERE rp.granted_role = r.role_name 
        AND grantee IN ((SELECT USERNAME FROM DBA_USERS WHERE ORACLE_MAINTAINED = 'N') 
                       UNION (SELECT ROLE FROM DBA_ROLES WHERE ORACLE_MAINTAINED = 'N'))
        AND granted_role IN (SELECT ROLE FROM DBA_ROLES WHERE ORACLE_MAINTAINED = 'N') 
        START WITH granted_role IN (SELECT DISTINCT role_name FROM obj_to_role) 
        CONNECT BY granted_role = PRIOR grantee) 
      WHERE grantee_type = 'USER'
    )
    UNION
    (
      -- Direct Object grants to User
      SELECT I.OWNER, I.OBJECT_NAME, GRANTEE, 
        DECODE(privilege, 'SELECT', 'READ', 'REFERENCE', 'READ', 'INSERT', 'WRITE',
               'UPDATE', 'WRITE', 'DELETE', 'WRITE', privilege) privs 
      FROM DBA_TAB_PRIVS, DBA_USERS 
      WHERE GRANTEE = USERNAME 
      AND OWNER = I.OWNER 
      AND TYPE = 'TABLE' 
      AND TABLE_NAME = I.OBJECT_NAME
    ) 
    ORDER BY 2 DESC;
  END LOOP;
END;
```

**Procedure for `SP_OBJECT_DEPENDENCY_ANALYSIS`**

```
CREATE OR REPLACE PROCEDURE SP_OBJECT_DEPENDENCY_ANALYSIS (v_level NUMBER) IS
  TYPE typ IS RECORD (
    schema VARCHAR2(100),
    obj_type VARCHAR2(100),
    obj_name VARCHAR2(100),
    path VARCHAR2(5000)
  );
  TYPE array IS TABLE OF typ;
  l_data array;
  c SYS_REFCURSOR;
  l_errors NUMBER;
  l_errno NUMBER;
  l_msg VARCHAR2(4000);
  l_idx NUMBER;
  l_level NUMBER;
BEGIN
  l_level := v_level + 1;
  OPEN c FOR 
    WITH obj_list AS (
      SELECT owner schema_name, object_type, object_name 
      FROM DEPENDENT_ANALYSIS_BASELINE 
      WHERE depedncy_level = v_level
    ),
    fw_dep_objects AS (
      SELECT level lvl, owner, name, type, referenced_owner, referenced_name,
        referenced_type, SYS_CONNECT_BY_PATH(name, ' -> ') || ' -> ' || referenced_name PATH 
      FROM dba_dependencies
      START WITH (owner, CASE WHEN type = 'PACKAGE BODY' THEN 'PACKAGE' ELSE type END, name) 
        IN (SELECT schema_name, object_type, object_name FROM obj_list)
      CONNECT BY NOCYCLE (owner, type, name) = 
        ((PRIOR referenced_owner, PRIOR referenced_type, PRIOR referenced_name))
    ),
    bw_dep_objects AS (
      SELECT level lvl, owner, name, type, referenced_owner, referenced_name,
        referenced_type, SYS_CONNECT_BY_PATH(name, ' <- ') || ' <- ' || referenced_name PATH 
      FROM dba_dependencies
      START WITH (referenced_owner, CASE WHEN referenced_type = 'PACKAGE BODY' THEN 'PACKAGE' 
        ELSE referenced_type END, referenced_name) IN (SELECT schema_name, object_type, object_name FROM obj_list)
      CONNECT BY NOCYCLE (referenced_owner, referenced_type, referenced_name) = 
        ((PRIOR owner, PRIOR type, PRIOR name))
    )
    SELECT * FROM (
      (SELECT DISTINCT referenced_owner schema, referenced_type obj_type, 
        referenced_name obj_name, path FROM fw_dep_objects)
      UNION
      (SELECT DISTINCT owner schema, type obj_type, name obj_name, path 
       FROM bw_dep_objects)
    )
    WHERE schema IN (SELECT username FROM all_users WHERE oracle_maintained = 'N')
    ORDER BY obj_type;

  LOOP
    FETCH c BULK COLLECT INTO l_data LIMIT 100;
    BEGIN
      FORALL i IN 1..l_data.count SAVE EXCEPTIONS
        INSERT INTO DEPENDENT_ANALYSIS_BASELINE (
          owner, object_name, object_type, catagory, depedncy_level, project_need, comments
        ) 
        VALUES (
          l_data(i).schema, 
          l_data(i).obj_name,
          CASE WHEN l_data(i).obj_type = 'PACKAGE BODY' THEN 'PACKAGE' ELSE l_data(i).obj_type END,
          'level ' || l_level || ' dependency',
          l_level,
          '',
          'from dependency proc' || l_data(i).path
        );
    EXCEPTION
      WHEN OTHERS THEN
        l_errors := sql%bulk_exceptions.count;
        FOR i IN 1..l_errors LOOP
          l_errno := sql%bulk_exceptions(i).error_code;
          l_msg := SQLERRM(-l_errno);
          l_idx := sql%bulk_exceptions(i).error_index;
          UPDATE DEPENDENT_ANALYSIS_BASELINE 
          SET catagory1 = catagory1 || ', found in level' || l_level || ' dependent of ' || l_data(l_idx).path,
              comments1 = comments1 || ', from dependency proc exception ' || l_data(i).path
          WHERE owner = l_data(l_idx).schema 
          AND object_name = l_data(l_idx).obj_name 
          AND object_type = l_data(l_idx).obj_type;
        END LOOP;
    END;
    EXIT WHEN c%NOTFOUND;
  END LOOP;
  CLOSE c;
END;
```
