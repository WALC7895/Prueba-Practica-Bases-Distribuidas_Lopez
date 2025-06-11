# Prueba-Practica-Bases-Distribuidas_Lopez
# 📚 Esquemas de Replicación de Bases de Datos

![License](https://img.shields.io/badge/License-MIT-blue)
![MySQL](https://img.shields.io/badge/MySQL-8.x-orange)
![SQL Server](https://img.shields.io/badge/SQL%20Server-2019-red)
![MongoDB](https://img.shields.io/badge/MongoDB-6.0-brightgreen)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-blue)

Este repositorio documenta la implementación detallada de **cinco esquemas de replicación** de bases de datos, utilizando entornos Linux y Windows, tanto en la nube de Google Cloud como en instalaciones locales.

---

## 📋 Tabla de Contenidos

1. [Arquitectura General](#-arquitectura-general)
2. [Prerequisitos](#-prerequisitos)
3. [Estructura del Repositorio](#-estructura-del-repositorio)
4. [Esquemas de Replicación](#-esquemas-de-replicaci%C3%B3n)

   * [1. MySQL Activa/Pasiva (Maestro–Esclavo)](#1-mysql-activapasiva-maestroesclavo)
   * [2. SQL Server Snapshot / Transactional / Merge](#2-sql-server-snapshot--transactional--merge)
   * [3. MySQL Síncrona (InnoDB Cluster)](#3-mysql-s%C3%ADncrona-innodb-cluster)
   * [4. MongoDB Replica Set (Lectura/Escritura)](#4-mongodb-replica-set-lecturaescritura)
   * [5. Replicación Heterogénea (MySQL → PostgreSQL)](#5-replicaci%C3%B3n-heterog%C3%A9nea-mysql--postgresql)
5. [Flujo de Trabajo (CI/CD)](#-flujo-de-trabajo-cicd)
6. [Diagrama de Arquitectura](#-diagrama-de-arquitectura)
7. [Solución de Problemas](#-soluci%C3%B3n-de-problemas)
8. [Conclusiones](#-conclusiones)
9. [Licencia](#-licencia)

---

## 🏛 Arquitectura General

Cada esquema se despliega en:

* **Google Cloud Platform** (Compute Engine, VPC, Firewall) para MySQL y MongoDB.
* **Entorno local Windows** para SQL Server.
* **Sincronización entre SGBD** usando SymmetricDS.

```text
                             +------------+
             +---------------| MySQL-1    | (Master)          
             |               +------------+
             |                     |
             |             +-------------+
             |             | MySQL-2     | (Slave)
             |             +-------------+
  GCP VPC---+
             |             +--------------+
             |             | Mongo Primary|
             |             +--------------+
             |              /    \
             |   +-----------+      +-----------+
             |   |Mongo Sec. |      | Mongo Arb |
             |   +-----------+      +-----------+
             |
             +-------------------------------------------+
                           | SQL Server Local |          
                           +------------------+          
                                         
                                   +---------------+
                                   | SymmetricDS    |
                                   +---------------+
                                         /      \
                               +--------+        +--------+
                               | MySQL-Src       | Postgres|
                               +-----------------+---------+
```

---

## 🔧 Prerequisitos

| Herramienta                 | Versión mínima | Notas                                |
| --------------------------- | -------------- | ------------------------------------ |
| Google Cloud SDK (`gcloud`) | 380.0.0        | Autenticación y despliegue opcional  |
| Ubuntu                      | 22.04 LTS      | MySQL, MongoDB, PostgreSQL           |
| MySQL Server                | 8.x            | Con soporte de Group Replication     |
| MySQL Shell                 | 8.x            | Gestión de InnoDB Cluster            |
| MongoDB                     | 6.0            | Replica Set                          |
| PostgreSQL                  | 15             | Destino de replicación heterogénea   |
| SQL Server                  | 2019           | Developer/Standard/Enterprise        |
| SSMS                        | 18.x           | Gestión de Replicación SQL Server    |
| SymmetricDS                 | 3.13.x         | Flujo heterogéneo MySQL → PostgreSQL |

---

## 📁 Estructura del Repositorio

```bash
/ (root)
├── 01-mysql-activa-pasiva/       # Maestro-Esclavo MySQL en GCP
│   ├── maestro/                  # Scripts & configs maestro
│   └── esclavo/                  # Scripts & configs esclavo
├── 02-sqlserver-replication/     # Snapshot, Transactional, Merge (SSMS)
│   ├── snapshot/                 # Guía paso a paso y scripts T-SQL
│   ├── transactional/            # Guía y scripts
│   └── merge/                    # Guía y scripts T-SQL
├── 03-mysql-innodb-cluster/      # InnoDB Cluster (3 nodos)
│   ├── node1/                    # Configuración y bootstrap
│   ├── node2/                    # Configuración y join
│   └── node3/                    # Configuración y join
├── 04-mongodb-replica-set/       # Replica Set MongoDB
│   ├── primary/                  # Config y rs.initiate()
│   ├── secondary/                # Config
│   └── arbiter/                  # Config
├── 05-heterogenea-mysql-postgres/ # SymmetricDS
│   ├── mysql-src/                # Config engine.properties MySQL
│   └── postgres-dst/             # Config engine.properties PG
└── README.md                     # Este documento
```

---

## 📁 Esquemas de Replicación

### 1. MySQL Activa/Pasiva (Maestro–Esclavo)

1. **VMs**: `mysql-maestro`, `mysql-esclavo` en GCP (e2-micro, Ubuntu 22.04).
2. **Firewall**: regla `allow-mysql-internal` (TCP 3306, rango `10.128.0.0/9`).
3. **Instalación**: `sudo apt install mysql-server` en ambas VMs.
4. **Maestro** (`mysqld.cnf`):

   ```ini
   server-id=1
   log-bin=mysql-bin
   binlog-do-db=DemoDB
   bind-address=0.0.0.0
   ```
5. **Usuario replicación**:

   ```sql
   CREATE USER 'replicador'@'%' IDENTIFIED BY 'R3plica!2024';
   GRANT REPLICATION SLAVE ON *.* TO 'replicador'@'%';
   FLUSH PRIVILEGES;
   ```
6. **Estado maestro**:

   ```sql
   FLUSH TABLES WITH READ LOCK;
   SHOW MASTER STATUS;
   ```
7. **Esclavo** (`mysqld.cnf`):

   ```ini
   server-id=2
   relay-log=relay-log
   bind-address=0.0.0.0
   ```
8. **Conectar esclavo**:

   ```sql
   CHANGE MASTER TO MASTER_HOST='IP-maestro', MASTER_USER='replicador', MASTER_PASSWORD='R3plica!2024', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=154;
   START SLAVE;
   SHOW SLAVE STATUS\G;
   ```
9. **Verificación**: insertar en maestro → `SELECT` en esclavo.

---

### 2. SQL Server Snapshot / Transactional / Merge

*Ejecutado exclusivamente en Windows con SSMS en dos instancias locales.*

#### 2.1 Preparación

* Instalar **SQL Server 2019** y **SSMS 18.x** en `Publisher` y `Subscriber`.
* Habilitar **TCP/IP** en SQL Server Configuration Manager (puerto 1433).
* Crear regla en Windows Firewall (TCP 1433).
* Asegurar que **SQL Server Agent** esté en **Automatic** y arrancado.
* Crear DB `DemoDB` con tablas:

  ```sql
  CREATE TABLE T_Snapshot(...);
  CREATE TABLE T_Transactional(...);
  CREATE TABLE T_Merge(...);
  ```

#### 2.2 Snapshot

1. Configure Distributor → Snapshot Publication (`Pub_Snapshot`).
2. Create Push Subscription to `DemoDB` on Subscriber.
3. Run Snapshot Agent.
4. Test:

   ```sql
   INSERT INTO T_Snapshot VALUES('Instant');
   SELECT * FROM T_Snapshot;  -- en Subscriber
   ```

#### 2.3 Transactional

1. Create Transactional Publication (`Pub_Transactional`).
2. Create Pull Subscription.
3. Test:

   ```sql
   INSERT INTO T_Transactional VALUES('Txn');
   SELECT * FROM T_Transactional;  -- en Subscriber
   ```

#### 2.4 Merge

1. Create Merge Publication (`Pub_Merge`).
2. Create Merge Subscription.
3. Test bidirectional:

   ```sql
   INSERT INTO T_Merge VALUES('From Pub');  -- Publisher
   INSERT INTO T_Merge VALUES('From Sub');  -- Subscriber
   -- Run Merge Agent
   SELECT * FROM T_Merge;  -- ambos lados
   ```

---

### 3. MySQL Síncrona (InnoDB Cluster)

1. **VMs**: 3 nodos Ubuntu con MySQL 8.x y MySQL Shell.
2. **Configurar** `mysqld.cnf` (GTID, plugins, group seeds).
3. **Bootstrap** en nodo1:

   ```js
   dba.createCluster('miCluster');
   ```
4. **Join** nodos 2/3:

   ```js
   cluster.addInstance('root@IPnodo2:3306');
   ```
5. **Verificar**:

   ```js
   cluster.status();  // todos ONLINE
   ```
6. **Test**: insertar en nodo1 → seleccionar en nodo2.

---

### 4. MongoDB Replica Set (Lectura/Escritura)

1. **VMs**: primary, secondary, arbiter Ubuntu.
2. **Instalar** MongoDB 6.0 y configurar `rsCloud` en `mongod.conf`.
3. **Iniciar Replica Set**:

   ```js
   rs.initiate({members:[...]});
   ```
4. **Verificar**: `rs.status()`.
5. **Test**:

   ```js
   db.messages.insert({msg:'hello'});  // primary
   db.getMongo().setReadPref('secondary'); db.messages.find();
   ```

---

### 5. Replicación Heterogénea (MySQL → PostgreSQL)

1. **VMs**: `mysql-src`, `postgres-dst`.
2. **Instalar** MySQL 8.x y PostgreSQL 15.
3. **Instalar** SymmetricDS 3.13.x.
4. **Configurar** `mysql.properties` y `postgres.properties`.
5. **Crear** tabla prueba en MySQL.
6. **Arrancar** servidores SymmetricDS y registrar nodos.
7. **Test**: insertar en MySQL → consultar en PostgreSQL.

---

## 🔄 Flujo de Trabajo (CI/CD)

* **GitHub Actions**: pipelines para validar cambios en scripts y configuraciones.
* **Lint**: `.sql` y `.conf`.
* **Docs**: generar site con MkDocs o GitHub Pages.

---

## 📐 Diagrama de Arquitectura

![](docs/architecture-diagram.png)

> Puedes encontrar Diagrama en `docs/architecture-diagram.drawio`.

---

## 🛠 Solución de Problemas

* **Slave\_IO\_Running = No** → revisar conectividad y credenciales.
* **Merge conflicts** → ajustar **conflict resolution** en Publication.
* **Replica Set not PRIMARY** → verificar `bindIp` y puertos.
* **SymmetricDS errors** → ver logs en `symmetric/log/`.

---

## 📝 Conclusiones

Este repositorio demuestra la configuración y validación de múltiples estrategias de replicación:

* **Alta disponibilidad** vs **consistencia eventual**.
* **Sincronización fuerte** (InnoDB Cluster) vs **asincrónica** (Maestro-Esclavo).
* Integración entre distintos motores (heterogénea).

---

## 🥇 Licencia

Este proyecto está licenciado bajo la **MIT License**. Revisa el archivo [LICENSE](LICENSE).
