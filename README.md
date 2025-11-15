# Oracle CDB Duplication & Data Guard Administration

![Oracle](https://img.shields.io/badge/Oracle_Database-21c-F80000?style=for-the-badge&logo=oracle)
![Docker](https://img.shields.io/badge/Docker-20.10-2496ED?style=for-the-badge&logo=docker)
![RMAN](https://img.shields.io/badge/Oracle_RMAN-Backup_&_Recovery-F80000?style=for-the-badge&logo=oracle)

This repository contains the work for a database administration project from **ISGA - Marrakech** (2024-2025), focusing on Oracle's multitenant architecture.

The project demonstrates two key pillars of modern Oracle database administration:
1.  **Practical CDB Duplication:** A step-by-step guide on performing an active duplication of a Container Database (CDB) using **RMAN** and **Docker**[cite: 130].
2.  [cite_start]**Theoretical Review of Data Guard:** An analysis of Oracle's high-availability and disaster recovery solution, **Data Guard**[cite: 130].

## 1. Practical Guide: Active CDB Duplication with RMAN

[cite_start]This part of the project provides a complete workflow for duplicating a source container database (`CDB1`) into a new target database (`CDB2`) using an active RMAN duplication[cite: 185, 224]. The entire process is containerized using Docker to ensure a clean, reproducible environment.

### ⚙️ Environment

* **Software:** Docker Desktop
* [cite_start]**Image:** `container-registry.oracle.com/database/enterprise:21.3.0.0` [cite: 234]
* [cite_start]**Source DB (Primary):** `oracle-cdb1` [cite: 237]
* [cite_start]**Target DB (Auxiliary):** `oracle-cdb2` [cite: 513]

### 📋 High-Level Workflow

Here is the summarized step-by-step process detailed in the report:

1.  **Docker Setup:**
    * [cite_start]Create a custom Docker network (`oracle-net`) for container communication[cite: 227].
    * [cite_start]Pull the Oracle 21c database image from the Oracle Container Registry[cite: 234].

2.  **Launch Source DB (`CDB1`):**
    * [cite_start]Start the `oracle-cdb1` container, exposing ports and setting environment variables for `ORACLE_SID` (`CDB1`), `ORACLE_PDB` (`PDB1`), and password [cite: 237-245].
    * [cite_start]Wait for the database to be fully created and ready[cite: 258].

3.  **Configure Source DB:**
    * [cite_start]Connect to `CDB1` using `sqlplus / as sysdba`[cite: 275].
    * [cite_start]Enable `ARCHIVELOG` mode, which is a prerequisite for RMAN duplication[cite: 302].
    * [cite_start]Enable `FORCE LOGGING`[cite: 308].
    * [cite_start]Create a test user and a test table with sample data to verify the duplication later [cite: 341-366].
    * [cite_start]Configure the `tnsnames.ora` [cite: 393-415] [cite_start]and `listener.ora` [cite: 420-434] files to allow network connections to both `CDB1` and the future `CDB2`.

4.  **Launch & Prepare Target DB (`CDB2`):**
    * [cite_start]Start the `oracle-cdb2` container in the same Docker network[cite: 513].
    * [cite_start]**Crucially,** once the DB is ready [cite: 527][cite_start], connect to it and **shut it down**[cite: 726].
    * [cite_start]Restart the instance in `NOMOUNT` mode[cite: 729]. This is required for RMAN to create a new control file.
    * [cite_start]Copy the `tnsnames.ora` [cite: 582-635] [cite_start]and `listener.ora` [cite: 637-658] configurations from the source.
    * [cite_start]Create a basic `initCDB2.ora` parameter file that defines the new `db_name` and, most importantly, the `db_file_name_convert` and `log_file_name_convert` parameters [cite: 740-751]. These tell RMAN how to rename file paths from the source (`.../CDB1/`) to the target (`.../CDB2/`).
    * [cite_start]Create the required directory structure for the new database files[cite: 756, 757].

5.  **Perform RMAN Duplication:**
    * [cite_start]Connect to the **source container** (`oracle-cdb1`)[cite: 766].
    * [cite_start]Launch RMAN, connecting to the source DB as `TARGET` and the target (auxiliary) DB as `AUXILIARY` over the network[cite: 775, 780].
    * [cite_start]Execute the `DUPLICATE DATABASE TO CDB2 FROM ACTIVE DATABASE` command[cite: 785].
    * RMAN handles the entire process: copying files, creating the new control file, and recovering the database.

6.  **Verify Duplication & Independence:**
    * [cite_start]Connect to the new `CDB2` database and open all its PDBs [cite: 804-807, 812].
    * Log into the `PDB1` on `CDB2` and run `SELECT * FROM test_duplication;`. [cite_start]The 3 records created on the source will be present [cite: 815-817].
    * [cite_start]**Test Independence:** Insert a new, 4th record into the table on `CDB2`[cite: 820].
    * [cite_start]Verify this 4th record exists on `CDB2` but **does not** exist on `CDB1`, proving the duplication was successful and the databases are now independent[cite: 829, 834].

---

## 2. Theoretical Review: Oracle Data Guard

[cite_start]The second part of the report is a theoretical study of Oracle Data Guard, Oracle's premier solution for high-availability (HA) and disaster recovery (DR) [cite: 836-840].

### Key Concepts

* [cite_start]**Primary Database:** The main production database that serves user traffic[cite: 853].
* [cite_start]**Standby Database:** One or more copies of the primary database[cite: 857].
    * [cite_start]**Physical Standby:** A block-for-block identical copy, maintained by applying Redo data[cite: 857].
    * [cite_start]**Logical Standby:** A logically identical copy, maintained by converting Redo data into SQL and re-executing it[cite: 858].

### Core Processes

The Data Guard architecture relies on several key background processes to move data:

* [cite_start]**`LGWR` / `LNS` (Log Writer / Log Network Server):** On the primary database, these processes capture Redo data and transmit it to the standby site[cite: 905, 911].
* [cite_start]**`RFS` (Remote File Server):** On the standby database, this process receives the Redo data from the network[cite: 918].
* [cite_start]**`MRP` (Managed Recovery Process):** On a physical standby, this process applies the received Redo logs to the database files[cite: 922].

### Data Protection Modes

Data Guard offers three distinct modes to balance performance and data protection:

1.  **Maximum Performance (Default):**
    * [cite_start]Uses **asynchronous** (`ASYNC`) Redo transport[cite: 933].
    * The primary DB commits transactions immediately, without waiting for the standby.
    * [cite_start]Offers the best performance but allows for minimal data loss if the primary site fails[cite: 932, 934].

2.  **Maximum Availability:**
    * Uses **synchronous** (`SYNC`) Redo transport.
    * The primary DB waits for confirmation that the Redo data is received by the standby, but *will not* stop if the standby is unavailable.
    * [cite_start]Prioritizes a "zero data loss" goal without sacrificing availability[cite: 932].

3.  **Maximum Protection:**
    * [cite_start]Uses **synchronous** (`SYNC`) Redo transport[cite: 929].
    * [cite_start]Guarantees **zero data loss**[cite: 928].
    * [cite_start]If the primary database cannot send its Redo data to at least one standby, **it will shut down** to prevent a data-loss scenario[cite: 928].

### Operations

* **Switchover:** A **planned** role reversal between a primary and standby database, typically for maintenance. [cite_start]This involves no data loss[cite: 940].
* [cite_start]**Failover:** An **unplanned** transition where a standby database is promoted to become the new primary after the original primary site fails[cite: 956].

## 👥 Authors

This project was a collaborative effort by:

* [cite_start]**Younes** [cite: 133]
* [cite_start]**Amine I.** [cite: 134]
* [cite_start]**Amine J.** [cite: 135]
* [cite_start]**Soufiyan** [cite: 136]

Special thanks to our professor, **Mr. [cite_start]Snineh**, for his guidance and support throughout this project[cite: 131, 152].
