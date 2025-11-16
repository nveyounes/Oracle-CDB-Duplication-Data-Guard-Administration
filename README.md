# Oracle CDB Duplication & Data Guard Administration

![Oracle](https://img.shields.io/badge/Oracle_Database-21c-F80000?style-for-the-badge&logo=oracle)
![Docker](https://img.shields.io/badge/Docker-20.10-2496ED?style-for-the-badge&logo=docker)
![RMAN](https://img.shields.io/badge/Oracle_RMAN-Backup_&_Recovery-F80000?style-for-the-badge&logo=oracle)

This repository contains the work for a database administration project from **ISGA - Marrakech** (2024-2025), focusing on Oracle's multitenant architecture.

The project demonstrates two key pillars of modern Oracle database administration:
1.  **Practical CDB Duplication:** A step-by-step guide on performing an active duplication of a Container Database (CDB) using **RMAN** and **Docker**.
2.  **Theoretical Review of Data Guard:** An analysis of Oracle's high-availability and disaster recovery solution, **Data Guard**.

## 1. Practical Guide: Active CDB Duplication with RMAN

This part of the project provides a complete workflow for duplicating a source container database (`CDB1`) into a new target database (`CDB2`) using an active RMAN duplication. The entire process is containerized using Docker to ensure a clean, reproducible environment.

### ⚙️ Environment

* **Software:** Docker Desktop
* **Image:** `container-registry.oracle.com/database/enterprise:21.3.0.0`
* **Source DB (Primary):** `oracle-cdb1`
* **Target DB (Auxiliary):** `oracle-cdb2`

### 📋 High-Level Workflow

Here is the summarized step-by-step process detailed in the report:

1.  **Docker Setup:**
    * Create a custom Docker network (`oracle-net`) for container communication.
    * Pull the Oracle 21c database image from the Oracle Container Registry.

2.  **Launch Source DB (`CDB1`):**
    * Start the `oracle-cdb1` container, exposing ports and setting environment variables for `ORACLE_SID` (`CDB1`), `ORACLE_PDB` (`PDB1`), and password.
    * Wait for the database to be fully created and ready.

3.  **Configure Source DB:**
    * Connect to `CDB1` using `sqlplus / as sysdba`.
    * Enable `ARCHIVELOG` mode, which is a prerequisite for RMAN duplication.
    * Enable `FORCE LOGGING`.
    * Create a test user and a test table with sample data to verify the duplication later.
    * Configure the `tnsnames.ora` and `listener.ora` files to allow network connections to both `CDB1` and the future `CDB2`.

4.  **Launch & Prepare Target DB (`CDB2`):**
    * Start the `oracle-cdb2` container in the same Docker network.
    * **Crucially,** once the DB is ready, connect to it and **shut it down**.
    * Restart the instance in `NOMOUNT` mode. This is required for RMAN to create a new control file.
    * Copy the `tnsnames.ora` and `listener.ora` configurations from the source.
    * Create a basic `initCDB2.ora` parameter file that defines the new `db_name` and, most importantly, the `db_file_name_convert` and `log_file_name_convert` parameters. These tell RMAN how to rename file paths from the source (`.../CDB1/`) to the target (`.../CDB2/`).
    * Create the required directory structure for the new database files.

5.  **Perform RMAN Duplication:**
    * Connect to the **source container** (`oracle-cdb1`).
    * Launch RMAN, connecting to the source DB as `TARGET` and the target (auxiliary) DB as `AUXILIARY` over the network.
    * Execute the `DUPLICATE DATABASE TO CDB2 FROM ACTIVE DATABASE` command.
    * RMAN handles the entire process: copying files, creating the new control file, and recovering the database.

6.  **Verify Duplication & Independence:**
    * Connect to the new `CDB2` database and open all its PDBs.
    * Log into the `PDB1` on `CDB2` and run `SELECT * FROM test_duplication;`. The 3 records created on the source will be present.
    * **Test Independence:** Insert a new, 4th record into the table on `CDB2`.
    * Verify this 4th record exists on `CDB2` but **does not** exist on `CDB1`, proving the duplication was successful and the databases are now independent.

---

## 2. Theoretical Review: Oracle Data Guard

The second part of the report is a theoretical study of Oracle Data Guard, Oracle's premier solution for high-availability (HA) and disaster recovery (DR).

### Key Concepts

* **Primary Database:** The main production database that serves user traffic.
* **Standby Database:** One or more copies of the primary database.
    * **Physical Standby:** A block-for-block identical copy, maintained by applying Redo data.
    * **Logical Standby:** A logically identical copy, maintained by converting Redo data into SQL and re-executing it.

### Core Processes

The Data Guard architecture relies on several key background processes to move data:

* **`LGWR` / `LNS` (Log Writer / Log Network Server):** On the primary database, these processes capture Redo data and transmit it to the standby site.
* **`RFS` (Remote File Server):** On the standby database, this process receives the Redo data from the network.
* **`MRP` (Managed Recovery Process):** On a physical standby, this process applies the received Redo logs to the database files.

### Data Protection Modes

Data Guard offers three distinct modes to balance performance and data protection:

1.  **Maximum Performance (Default):**
    * Uses **asynchronous** (`ASYNC`) Redo transport.
    * The primary DB commits transactions immediately, without waiting for the standby.
    * Offers the best performance but allows for minimal data loss if the primary site fails.

2.  **Maximum Availability:**
    * Uses **synchronous** (`SYNC`) Redo transport.
    * The primary DB waits for confirmation that the Redo data is received by the standby, but *will not* stop if the standby is unavailable.
    * Prioritizes a "zero data loss" goal without sacrificing availability.

3.  **Maximum Protection:**
    * Uses **synchronous** (`SYNC`) Redo transport.
    * Guarantees **zero data loss**.
    * If the primary database cannot send its Redo data to at least one standby, **it will shut down** to prevent a data-loss scenario.

### Operations

* **Switchover:** A **planned** role reversal between a primary and standby database, typically for maintenance. This involves no data loss.
* **Failover:** An **unplanned** transition where a standby database is promoted to become the new primary after the original primary site fails.

## 👥 Authors

This project was a collaborative effort by:

* **Younes**
* **Amine I.**
* **Amine J.**
* **Soufiyan**

Special thanks to our professor, **Mr. Snineh**, for his guidance and support throughout this project.
