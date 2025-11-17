# UTS — Sistem Terdistribusi

**Nama:** _MUHAMMAD VICKY SAPUTRA_  
**NIM:** _245410079_  
**Kelas:** _IF_

## 1. Teorema CAP dan BASE

### Teorema CAP (ringkas)
CAP menyatakan bahwa pada sistem terdistribusi dengan kemungkinan terjadi *network partition*, tidak mungkin menjamin **Consistency (C)**, **Availability (A)**, dan **Partition Tolerance (P)** sekaligus. Saat partisi terjadi, Anda harus memilih:
- **C + P (CP)** → konsistensi diutamakan (ketersediaan dapat terganggu).
- **A + P (AP)** → ketersediaan diutamakan (konsistensi bersifat eventual).

**Definitions**
- **Consistency (C):** setiap read mendapat data yang terbaru / "satu sumber kebenaran".
- **Availability (A):** setiap request mendapat respon (sistem tidak menolak).
- **Partition Tolerance (P):** sistem tetap bekerja walau sebagian node tidak saling terhubung.

### BASE (ringkas)
BASE adalah pola desain yang sering dipakai pada sistem yang memilih **Availability** (AP):
- **Basically Available:** sistem tetap responsif.
- **Soft state:** state bisa berubah sementara karena replikasi/propagasi.
- **Eventual consistency:** data akan konsisten akhirnya.

### Hubungan CAP & BASE
- CAP adalah teori trade-off; BASE adalah pendekatan praktis bila memilih AP (mengorbankan konsistensi kuat demi availability dan partition tolerance).
- Sistem yang memilih CP biasanya memakai model konsistensi kuat (ACID), sedangkan sistem BASE menerima konsistensi *eventual*.

### Contoh (berdasarkan pengalaman)
- **CP (mis. sistem pembayaran):** saldo harus konsisten → gunakan synchronous replication atau locking terdistribusi. Saat partisi, transaksi mungkin ditolak (menjaga konsistensi).
- **AP (mis. feed sosial / catalog):** tetap memberi respon cepat walau data sedikit stale → gunakan caching + eventual consistency (BASE).

---

## 2. GraphQL & Komunikasi Antar-Proses (IPC) pada Sistem Terdistribusi

**Inti:** GraphQL biasanya ditempatkan sebagai *API gateway* yang mengorkestrasi panggilan ke berbagai microservice. Ia menerima satu query dari klien lalu memanggil banyak service (HTTP/gRPC/queue), menggabungkan hasil, dan mengembalikan satu response. Dengan demikian GraphQL memfasilitasi komunikasi antar-proses (IPC) — tetapi tidak menggantikan protokol IPC.

**Manfaat:**
- Aggregation (menggabungkan hasil dari banyak service)
- Batching (mengurangi N+1 dengan DataLoader)
- Abstraction (menyembunyikan heterogenitas backend)
- Fine-grained fetching (klien hanya minta field yang diperlukan)

### Diagram (box / kotak) — Gunakan ini di GitHub (Mermaid)
> Jika Anda menaruh ini di `README.md` di GitHub, GitHub akan merender diagram Mermaid otomatis.

```mermaid
flowchart LR
    Client([Client Application])
    GraphQL([GraphQL API Gateway])
    subgraph Microservices
      UserSvc([User Service<br/>REST/gRPC])
      ProductSvc([Product Service<br/>REST/gRPC])
      InventorySvc([Inventory Service<br/>DB/Cache])
    end

    Client --> GraphQL
    GraphQL --> UserSvc
    GraphQL --> ProductSvc
    GraphQL --> InventorySvc

    UserSvc --> GraphQL
    ProductSvc --> GraphQL
    InventorySvc --> GraphQL

    GraphQL --> Client
 ```

 ---
# 3. Streaming Replication PostgreSQL Menggunakan Docker / Docker Compose
## 3.1 Tujuan & Gambaran Umum

Streaming replication adalah metode sinkronisasi data di PostgreSQL di mana primary server mengirimkan perubahan melalui Write-Ahead Log (WAL) ke replica server secara real time.

Manfaat:
- High Availability
- Backup real-time
- Read scaling
- Disaster recovery
- Primary = baca + tulis
- Replica = read-only

## 3.2 Gambaran Arsitektur
```mermaid
flowchart LR
    App[Client / Application] -->|Read/Write| Primary[(Primary PostgreSQL)]
    Primary -->|Streaming WAL| Replica[(Replica PostgreSQL)]

    subgraph WAL_Replication
        Primary ---|Send WAL Records| Replica
    end
```
### Pembahasan Arsitektur Streaming Replication

1. **Client / Application**  
   Klien bertugas mengirimkan permintaan baca dan tulis ke server **Primary**. Semua interaksi awal dilakukan melalui node primary.

2. **Primary Database**  
   - Menangani seluruh operasi *write* (INSERT, UPDATE, DELETE).  
   - Setiap perubahan tidak langsung ditulis ke file data, tetapi dicatat lebih dulu pada **Write-Ahead Log (WAL)**.  
   - WAL ini kemudian dikirim secara real-time ke Replica untuk menjaga sinkronisasi.

3. **WAL Replication**  
   WAL berfungsi sebagai catatan semua perubahan pada database.  
   Mekanisme streaming replication akan mengalirkan log WAL ini terus-menerus ke Replica.

4. **Replica Database**  
   - Hanya menerima operasi baca (read-only).  
   - Menerapkan perubahan dari WAL yang dikirim oleh Primary.  
   - Secara otomatis mengikuti kondisi terbaru Primary sehingga datanya selalu up-to-date.

---

### **Kesimpulan Arsitektur**

Arsitektur Streaming Replication memungkinkan dua server database berjalan sinkron secara otomatis:

- **Primary** menangani operasi *write*.  
- **Replica** menangani *read-only*.  
- **Write-Ahead Log (WAL)** menjadi mekanisme utama sinkronisasi data.

Dengan cara ini, sistem menjadi lebih aman, cepat, dan mendukung high availability.


Arsitektur ini membuat dua server database berjalan sinkron secara otomatis:
- Primary untuk write
- Replica untuk read
- WAL sebagai mekanisme sinkronisas

### `primary/conf.d/postgresql.conf`

File ini dipakai untuk mengatur perilaku PostgreSQL pada sisi **Primary server**, terutama untuk mengaktifkan fitur **Streaming Replication**.  
Tanpa konfigurasi di file ini, **Replica tidak akan bisa menerima WAL (Write-Ahead Log)** dan tidak akan pernah sinkron dengan Primary.

---

### **Isi File Konfigurasi**

```conf
listen_addresses = '*'
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = '64MB'

synchronous_commit = on
synchronous_standby_names = 'replica1'
```
### Penjelasan Setiap Baris Konfigurasi

#### **1. `listen_addresses = '*'`**
Mengizinkan PostgreSQL menerima koneksi dari semua IP.  
Karena Replica berjalan di container berbeda, maka Primary harus terbuka untuk menerima koneksi.

#### **2. `wal_level = replica`**
Mengaktifkan level WAL yang dibutuhkan untuk replikasi.  
Jika tidak diatur, WAL tidak akan dikirim ke Replica.

#### **3. `max_wal_senders = 10`**
Mengatur maksimal jumlah proses pengirim WAL.  
Replica membutuhkan minimal 1 WAL sender untuk menerima stream dari Primary.

#### **4. `max_replication_slots = 10`**
Menyediakan slot bagi setiap Replica agar tidak tertinggal WAL.  
Tanpa slot, jika WAL terhapus terlalu cepat, replikasi dapat gagal.

#### **5. `wal_keep_size = '64MB'`**
Menjamin WAL tersimpan minimal 64MB agar tidak cepat terhapus.  
Hal ini penting agar Replica punya waktu cukup untuk menyalin WAL tanpa ketinggalan.

#### **6. `synchronous_commit = on`**
Transaksi di Primary **baru dianggap berhasil** setelah Replica menerima WAL.  
Ini membuat replikasi menjadi **sinkron** (lebih aman pada data penting).

#### **7. `synchronous_standby_names = 'replica1'`**
Menentukan nama standby (Replica) yang wajib ikut sinkron.  
Apabila namanya cocok, Primary akan menunggu sampai Replica menerima WAL sebelum commit.

---

### **Kesimpulan**

File konfigurasi `postgresql.conf` ini berfungsi untuk:

- Mengaktifkan fitur **Streaming Replication**
- Mengatur agar WAL dikirim ke Replica secara realtime
- Mengatur agar replikasi berjalan secara **sinkron** (bukan asynchronous)
- Menjamin data Primary dan Replica tetap sinkron
- Menjadi komponen penting untuk sistem **High Availability**
---
```sql
  CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replica_pass';
```
## 📝 Penjelasan Setiap Bagian Perintah SQL

### **1. `CREATE ROLE replicator`**
Perintah ini membuat sebuah role/user baru bernama **replicator**.  
User ini digunakan oleh Replica untuk terhubung ke Primary dan melakukan proses replikasi.

---

### **2. `WITH REPLICATION`**
Memberikan privilege **REPLICATION** kepada user.  
Privilege ini sangat penting karena memungkinkan user untuk:

- Mengambil *base backup* dari Primary menggunakan `pg_basebackup`
- Menerima aliran WAL (Write-Ahead Log) untuk replikasi
- Menjalankan sinkronisasi data secara realtime

Tanpa privilege ini, proses replikasi akan **gagal** karena Replica tidak memiliki izin mengambil WAL.

---

### **3. `LOGIN`**
Mengizinkan role ini untuk login ke server PostgreSQL.  
Jika opsi ini tidak diberikan, user tidak bisa digunakan untuk autentikasi ke Primary.

---

### **4. `PASSWORD 'replica_pass'`**
Memberikan password kepada user.  
Password ini dipakai oleh Replica dalam script `entrypoint.sh` ketika menghubungi Primary.

Password harus sama seperti yang ditentukan dalam konfigurasi Docker Compose.
```bash
#!/usr/bin/env bash
set -e

PRIMARY_HOST="primary"
PRIMARY_PORT=5432
REPL_USER="replicator"
REPL_PASSWORD="replica_pass"
PGDATA="/var/lib/postgresql/data"

echo "Waiting for primary..."
until pg_isready -h ${PRIMARY_HOST} -p ${PRIMARY_PORT} -U postgres >/dev/null 2>&1; do
  sleep 1
done

export PGPASSWORD=${REPL_PASSWORD}

rm -rf ${PGDATA}/*
pg_basebackup -h ${PRIMARY_HOST} -p ${PRIMARY_PORT} -D ${PGDATA} -U ${REPL_USER} -Fp -Xs -P -R

cat >> ${PGDATA}/postgresql.conf <<EOF
hot_standby = on
listen_addresses = '*'
EOF

exec postgres
```
