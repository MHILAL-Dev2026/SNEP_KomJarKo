# Analisis Pengiriman Paket Data (Packet Delivery Process)

Dokumen ini menyajikan analisis komprehensif mengenai mekanisme transmisi paket data lintas subnet pada arsitektur jaringan berbasis 1 router dan 5 host.

---

## 1. Pemetaan Arsitektur Jaringan

### A. Segmen Kiri (Subnet 192.168.1.0/24)
- **Host Pengirim:** `192.168.1.4` (Host 1)
- **Host Anggota Lain:**
  - `192.168.1.3` (Host 2)
  - `192.168.1.2` (Host 3)
- **Interface Gerbang (Default Gateway):** `192.168.1.1` (FastEthernet 0/0)

### B. Segmen Kanan (Subnet 192.168.2.0/24)
- **Host Tujuan:** `192.168.2.2` (Host 2)
- **Host Anggota Lain:**
  - `192.168.2.3` (Host 1)
- **Interface Gerbang (Default Gateway):** `192.168.2.1` (FastEthernet 0/1)

---

## 2. Diagram Alur Transmisi Paket (192.168.1.4 → 192.168.2.2)

```text
[Host 192.168.1.4] 
       │
       ▼ (Validasi Subnet & Resolusi ARP Gateway)
   [Switch0]
       │
       ▼
[Router Interface f0/0: 192.168.1.1]
       │
       ▼ (Pemeriksaan Routing Table: Directly Connected ke 192.168.2.0/24)
[Router Interface f0/1: 192.168.2.1]
       │
       ▼
   [Switch1]
       │
       ▼
[Host 192.168.2.2] (Paket Berhasil Diterima)
```

---

## 3. Tahapan Rinci Mekanisme Transmisi

### Fase 1: Evaluasi Alamat pada Host Sumber (`192.168.1.4`)
1. **Pemeriksaan Subnet Mask:** Host pengirim menetapkan target paket ke `192.168.2.2` dan melakukan operasi AND bitwise menggunakan subnet mask `255.255.255.0`.
2. **Keputusan Routing Lokal:** Karena segmen tujuan berbeda dari network lokal (`192.168.1.0/24`), paket diserahkan ke *Default Gateway* (`192.168.1.1`).
3. **Resolusi ARP & Enkapsulasi:** Host memeriksa cache ARP untuk memperoleh MAC address dari antarmuka router `f0/0`. Setelah alamat fisik didapat, frame Ethernet disusun menggunakan:
   - **IP Sumber:** `192.168.1.4`
   - **IP Tujuan:** `192.168.2.2`
   - **MAC Tujuan:** MAC Address antarmuka `f0/0` milik router.

---

### Fase 2: Pemrosesan oleh Router
1. **Penerimaan Frame:** `Switch0` meneruskan sinyal frame ke interface `f0/0` router.
2. **Dekapsulasi Layer 2:** Router melepas frame Ethernet untuk membaca informasi header Layer 3 (IP Header), mengenali IP tujuan `192.168.2.2`.
3. **Pencarian Tabel Rute:** Router meninjau tabel *routing* miliknya. Karena subnet `192.168.2.0/24` berstatus **Directly Connected** pada port `f0/1` (`192.168.2.1`), router langsung mengenali jalur tujuan tanpa memerlukan konfigurasi static route tambahan.
4. **Modifikasi Header & Re-enkapsulasi:**
   - Router menurunkan nilai **TTL (Time to Live)** sebanyak 1.
   - Router mengenkapsulasi ulang paket ke frame baru dengan MAC address tujuan milik host akhir (atau mengirim ARP request di segmen kanan jika belum ada).
   - Paket diteruskan keluar melalui port `f0/1`.

---

### Fase 3: Pengiriman ke Host Akhir (`192.168.2.2`)
1. **Distribusi Switch:** Frame melintasi port `f0/1` menuju `Switch1`.
2. **Penerusan Frame:** `Switch1` membaca MAC address tujuan dan meneruskan paket ke port yang terhubung ke `192.168.2.2`.
3. **Penyelesaian Transmisi:** Host `192.168.2.2` menerima frame, membuka lapisan enkapsulasi protokol hingga Layer 3/4, lalu memproses data yang dikirimkan.

---

## 4. Kesimpulan

Komunikasi data antar-subnet dalam topologi router tunggal dapat berjalan langsung tanpa penambahan rute statis (*static routing*). Hal ini dimungkinkan karena semua subnet yang terhubung secara fisik ke interface router yang aktif secara otomatis tercatat dalam tabel *routing* sebagai rute langsung (**Directly Connected**).