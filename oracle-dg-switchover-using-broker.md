
# Oracle Data Guard Switchover Procedure & Job Monitoring

---

## 1. Cek Parameter Recovery

```sql
SHOW PARAMETER recovery;
```

## 2. Pastikan Mount Point Direktori ARC Cukup

```bash
df -h
```

## 3. Cek Status Database Standby (Open Read Only)

```sql
SELECT open_mode FROM v$database;
-- Pastikan status: READ ONLY WITH APPLY
```

---

## 4. Gunakan Data Guard Broker

```bash
dgmgrl /
SET TIME ON;
```

### Login ke DGMGRL
```bash
CONNECT sys
Input Password : [masukkan password sys]
```

### Jika perlu buka 2 tab terminal untuk mode multi exec

```bash
-- Tab 1 & 2
SHOW CONFIGURATION;

SHOW DATABASE VERBOSE service_namedc;
SHOW DATABASE VERBOSE service_namedrc;

VALIDATE DATABASE service_namedc;
VALIDATE DATABASE service_namedrc;
VALIDATE NETWORK CONFIGURATION FOR ALL;
```

---

## 5. Switchover ke Primary Baru

```bash
SWITCHOVER TO service_namedc;
```

Tunggu proses selesai, lalu verifikasi kembali konfigurasi dan status:

```bash
SHOW CONFIGURATION;
```

---

## 6. Query untuk Monitoring Job Scheduler

### Query Lengkap Job Scheduler
```sql
SELECT owner AS schema_name,
       job_name,
       job_style,
       CASE WHEN job_type IS NULL THEN 'PROGRAM' ELSE job_type END AS job_type,
       CASE WHEN job_type IS NULL THEN program_name ELSE job_action END AS job_action,
       start_date,
       CASE WHEN repeat_interval IS NULL THEN schedule_name ELSE repeat_interval END AS schedule,
       last_start_date,
       next_run_date,
       state
FROM sys.all_scheduler_jobs
ORDER BY owner, job_name;
```

### Tambahan Format Kolom (opsional di SQL*Plus)
```sql
COL schema_name FORMAT a15;
COL job_name FORMAT a15;
COL last_start_date FORMAT a40;
COL next_run_date FORMAT a40;
```

### Query Job Non-SYS/SYSTEM
```sql
SELECT owner AS schema_name,
       job_name,
       job_style,
       CASE WHEN job_type IS NULL THEN 'PROGRAM' ELSE job_type END AS job_type,
       last_start_date,
       state
FROM sys.all_scheduler_jobs
WHERE owner NOT IN ('SYS', 'SYSTEM')
ORDER BY owner, job_name;
```

---

## 7. Catatan Tambahan

Jika belum dalam status read only, maka alter terlebih dahulu:

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
ALTER DATABASE OPEN READ ONLY;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT USING CURRENT LOGFILE; --atau
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT FROM SESSION;
```
DGMGRL> EDIT DATABASE 'standby_db' SET STATE='APPLY-OFF';
DGMGRL> EDIT DATABASE 'standby_db' SET STATE='APPLY-ON';

- Selalu verifikasi status database sebelum dan sesudah switchover.
- Pastikan tidak ada job penting yang berjalan saat switchover.
- Cek log file untuk troubleshooting jika terjadi error saat proses switchover.
