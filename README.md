# Database-Finance-on-VM
# Finance AI Database Foundation

## Overview

Project ini berfokus pada pembangunan database keuangan menggunakan PostgreSQL sebagai fondasi utama sebelum integrasi dengan AI Agent, API, Dashboard, maupun Telegram Bot.

Tujuan tahap ini adalah membangun database yang:

* Memiliki struktur data yang realistis
* Memiliki relasi antar tabel
* Memiliki data transaksi keuangan
* Memiliki view untuk analisis bisnis
* Siap diakses dari aplikasi lain
* Siap digunakan oleh AI Agent pada tahap berikutnya

---

# Architecture

```text
Windows (pgAdmin 4)
        │
        ▼
Ubuntu VM
        │
        ▼
PostgreSQL
        │
        ├── Tables
        ├── Views
        ├── Finance Data
        └── Database Users
```

---

# Objectives

Database ini dirancang untuk mensimulasikan aktivitas perusahaan selama 24 bulan.

Dataset yang akan dibangun:

| Entity          | Target Data |
| --------------- | ----------: |
| Customers       |         100 |
| Vendors         |          20 |
| Employees       |          50 |
| Invoices        |         500 |
| Payments        |        ±350 |
| Expenses        |        2000 |
| Journal Entries |        5000 |

---

# Environment

## Server

Ubuntu Server

## Database

PostgreSQL

## Client

pgAdmin 4

---

# PostgreSQL Installation

Update package repository:

```bash
sudo apt update
```

Install PostgreSQL:

```bash
sudo apt install postgresql postgresql-contrib -y
```

Verifikasi instalasi:

```bash
psql --version
```

Contoh hasil:

```text
psql (PostgreSQL) 16.x
```

---

# Create Database

Masuk ke PostgreSQL:

```bash
sudo -u postgres psql
```

Buat database:

```sql
CREATE DATABASE finance_ai;
```

Masuk ke database:

```sql
\c finance_ai
```

---

# Database Schema

Database terdiri dari 7 tabel utama.

## Customers

Menyimpan data pelanggan.

```sql
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    customer_code VARCHAR(20),
    customer_name VARCHAR(255),
    industry VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## Vendors

Menyimpan data supplier.

```sql
CREATE TABLE vendors (
    vendor_id SERIAL PRIMARY KEY,
    vendor_code VARCHAR(20),
    vendor_name VARCHAR(255),
    service_type VARCHAR(100)
);
```

---

## Employees

Menyimpan data karyawan.

```sql
CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    employee_name VARCHAR(255),
    department VARCHAR(100),
    salary NUMERIC(15,2)
);
```

---

## Invoices

Menyimpan tagihan kepada customer.

```sql
CREATE TABLE invoices (
    invoice_id SERIAL PRIMARY KEY,
    invoice_number VARCHAR(50),
    customer_id INTEGER REFERENCES customers(customer_id),
    invoice_date DATE,
    due_date DATE,
    amount NUMERIC(15,2),
    status VARCHAR(50)
);
```

---

## Payments

Menyimpan pembayaran invoice.

```sql
CREATE TABLE payments (
    payment_id SERIAL PRIMARY KEY,
    invoice_id INTEGER REFERENCES invoices(invoice_id),
    payment_date DATE,
    amount NUMERIC(15,2)
);
```

---

## Expenses

Menyimpan biaya operasional perusahaan.

```sql
CREATE TABLE expenses (
    expense_id SERIAL PRIMARY KEY,
    vendor_id INTEGER REFERENCES vendors(vendor_id),
    expense_date DATE,
    category VARCHAR(100),
    amount NUMERIC(15,2)
);
```

---

## Journal Entries

Menyimpan transaksi akuntansi.

```sql
CREATE TABLE journal_entries (
    journal_id SERIAL PRIMARY KEY,
    entry_date DATE,
    account_name VARCHAR(100),
    debit NUMERIC(15,2),
    credit NUMERIC(15,2),
    description TEXT
);
```

---

# Database Relationship

```text
Customers
    │
    ▼
Invoices
    │
    ▼
Payments


Vendors
    │
    ▼
Expenses
    │
    ▼
Journal Entries
```

---

# Finance Views

View digunakan untuk mempermudah analisis data.

## Revenue Monthly

```sql
CREATE OR REPLACE VIEW vw_revenue_monthly AS
SELECT
    DATE_TRUNC('month', invoice_date)::date AS month,
    COUNT(*) AS total_invoice,
    SUM(amount) AS revenue
FROM invoices
GROUP BY 1
ORDER BY 1;
```

---

## Payment Monthly

```sql
CREATE OR REPLACE VIEW vw_payment_monthly AS
SELECT
    DATE_TRUNC('month', payment_date)::date AS month,
    COUNT(*) AS total_payment,
    SUM(amount) AS cash_in
FROM payments
GROUP BY 1
ORDER BY 1;
```

---

## Expense Monthly

```sql
CREATE OR REPLACE VIEW vw_expense_monthly AS
SELECT
    DATE_TRUNC('month', expense_date)::date AS month,
    COUNT(*) AS total_expense,
    SUM(amount) AS expense
FROM expenses
GROUP BY 1
ORDER BY 1;
```

---

## Overdue Invoice

```sql
CREATE OR REPLACE VIEW vw_overdue_invoice AS
SELECT
    invoice_number,
    customer_id,
    invoice_date,
    due_date,
    amount
FROM invoices
WHERE status = 'Overdue';
```

---

## Top Customer

```sql
CREATE OR REPLACE VIEW vw_top_customer AS
SELECT
    c.customer_name,
    COUNT(i.invoice_id) AS total_invoice,
    SUM(i.amount) AS total_revenue
FROM customers c
JOIN invoices i
ON c.customer_id = i.customer_id
GROUP BY c.customer_name
ORDER BY total_revenue DESC;
```

---

## Profit Monthly

```sql
CREATE OR REPLACE VIEW vw_profit_monthly AS
WITH revenue AS (
    SELECT
        DATE_TRUNC('month', invoice_date)::date AS month,
        SUM(amount) AS revenue
    FROM invoices
    GROUP BY 1
),
expense AS (
    SELECT
        DATE_TRUNC('month', expense_date)::date AS month,
        SUM(amount) AS expense
    FROM expenses
    GROUP BY 1
)
SELECT
    r.month,
    r.revenue,
    COALESCE(e.expense,0) AS expense,
    r.revenue - COALESCE(e.expense,0) AS profit
FROM revenue r
LEFT JOIN expense e
ON r.month = e.month
ORDER BY r.month;
```

---

# PostgreSQL Network Configuration

Cari IP server:

```bash
hostname -I
```

atau

```bash
ip addr show
```

Contoh:

```text
192.168.1.100
```

---

## Enable External Connections

Edit:

```bash
sudo nano /etc/postgresql/*/main/postgresql.conf
```

Ubah:

```ini
listen_addresses='*'
```

---

## Configure pg_hba.conf

Edit:

```bash
sudo nano /etc/postgresql/*/main/pg_hba.conf
```

Tambahkan:

```ini
host all all 192.168.1.0/24 scram-sha-256
```

Sesuaikan subnet dengan jaringan masing-masing.

---

## Restart PostgreSQL

```bash
sudo systemctl restart postgresql
```

Verifikasi:

```bash
sudo systemctl status postgresql
```

---

# Database Users

## Application User

Membuat user yang akan digunakan oleh aplikasi eksternal.

```sql
CREATE USER finance_user
WITH PASSWORD 'Finance123!';
```

Berikan hak akses:

```sql
GRANT CONNECT
ON DATABASE finance_ai
TO finance_user;

GRANT USAGE
ON SCHEMA public
TO finance_user;

GRANT SELECT, INSERT, UPDATE
ON ALL TABLES IN SCHEMA public
TO finance_user;
```

Default privilege:

```sql
ALTER DEFAULT PRIVILEGES
IN SCHEMA public
GRANT SELECT, INSERT, UPDATE
ON TABLES
TO finance_user;
```

---

# pgAdmin 4 Connection

Cari IP server:

```bash
hostname -I
```

Contoh:

```text
192.168.1.100
```

Gunakan pada pgAdmin:

```text
Host      : 192.168.1.100
Port      : 5432
Database  : finance_ai
Username  : finance_user
Password  : Finance123!
```

---

# Verification

Cek database:

```sql
\dt
```

Cek view:

```sql
\dv
```

Cek koneksi:

```bash
ss -tulpn | grep 5432
```

Pastikan PostgreSQL mendengarkan pada:

```text
0.0.0.0:5432
```

atau

```text
192.168.x.x:5432
```

---

# Next Stage

Tahap berikutnya akan menggunakan database ini sebagai sumber data untuk:

* FastAPI Backend
* AI Agent
* Telegram Bot
* Web Dashboard

Pada tahap ini database telah selesai dibangun dan siap digunakan oleh sistem lain melalui koneksi PostgreSQL standar.
