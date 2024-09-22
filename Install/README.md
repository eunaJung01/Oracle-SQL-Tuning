# Install & Connect Oracle DB on an M1 Mac

### Contents

1. [References](#References)
2. [Docker Commands](#Docker-commands)
3. [Connect to Oracle DB from DataGrip](#Connect-to-Oracle-DB-from-DataGrip)
4. [+) CDB vs. PDB ?](#cdb-vs-pdb)

<br/>
<br/>

## References

- ⭐️ https://www.youtube.com/watch?v=uxvoMhkKUPE ⭐️
- https://github.com/oracle/docker-images
- https://www.jetbrains.com/help/datagrip/oracle.html

<br/>
<br/>

## Docker Commands

Directory path : `~/.../docker-images/OracleDatabase/SingleInstance/dockerfiles`

```bash
docker run -d --name oracle19 -e ORACLE_PWD=mypassword1 -p 1521:1521 oracle/database:19.3.0-ee
```

<br>

### Check Docker Status

```bash
docker ps
```

<p align="center"><img width="1200" alt="docker_ps" src="https://github.com/user-attachments/assets/084cc802-03c3-4079-a3d3-2252207ca10f">

<br/>
<br/>
<br/>

## Connect to Oracle DB from DataGrip

### Oracle 19c CDB

- Connection type : `Service name`
- Host : `localhost`
- Port : `1521`
- Service : `ORCLCDB`
- User : `SYS as SYSDBA`
- Password : `mypassword1`

<p align="center"><img width="800" alt="oracle_cdb_connection" src="https://github.com/user-attachments/assets/57a07cce-1fdf-46d1-8812-bd240dc3adee">

<br/>
<br/>

### Oracle 19c PDB

- Connection type : `Service name`
- Host : `localhost`
- Port : `1521`
- Service : `ORCLPDB1`
- User : `SYS as SYSDBA`
- Password : `mypassword1`

<p align="center"><img width="800" alt="oracle_pdb_connection" src="https://github.com/user-attachments/assets/f85244c4-3c24-4935-bc08-7873423143b7">

<br/>
<br/>

### Oracle 19c PDB - Create User

```oracle
CREATE USER euna
    IDENTIFIED BY mypassword1;

GRANT RESOURCE, CONNECT, DBA TO euna;
```

<br/>

- Connection type : `Service name`
- Host : `localhost`
- Port : `1521`
- Service : `ORCLPDB1`
- User : `euna`
- Password : `mypassword1`

<p align="center"><img width="800" alt="oracle_pdb_euna_connection" src="https://github.com/user-attachments/assets/6e04efcd-5701-4495-a722-895692f3cad5">

<br/>
<br/>

\\^0^/
<p align="center"><img width="350" alt="\^0^/" src="https://github.com/user-attachments/assets/2c75e9bf-81bd-4195-98dc-112ca6a79032">

<br/>
<br/>
<br/>

<h2 id="cdb-vs-pdb">+) CDB vs. PDB ?</h2>

https://docs.oracle.com/en/database/oracle/oracle-database/21/cncpt/CDBs-and-PDBs.html

<br/>

<p align="center"><img width="700" alt="cdb_pdb" src="https://github.com/user-attachments/assets/c954b518-e770-4d00-9dbc-ede28b7a7af9">

<br/>
<br/>

### Container Database, CDB

A `CDB` includes zero, one, or many customer-created pluggable databases (PDBs) and application containers.

<br/>

### Pluggable Database, PDB

A `PDB` is a portable collection of schemas, schema objects, and nonschema objects  
that **appears logically to an client application as a separate database.**

<br/>

### SYS User

Every PDB is owned by `SYS`, regardless of which user created the PDB.

`SYS` is a **common user** in the CDB,    
which means that this user that has the same identity in the root and in every existing and future PDB within the CDB.
