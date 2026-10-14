# Analisis Mekanisme Pengiriman Paket Data Jaringan (Static Routing)

Dokumen ini memaparkan penelusuran mendalam terkait alur dan tahapan transmisi paket data (*Packet Delivery Process*) antar-segmen jaringan menggunakan topologi 2 Router dan 5 Host dengan penerapan rute statis (*Static Routing*).

---

## 1. Topologi dan Alokasi Pengalamatan

### A. Segmen Jaringan Lokal Kiri (Subnet `192.168.1.0/24`)
* **Host Pengirim:** `192.168.1.4` (Host 1)
* **Host Lain dalam Subnet:**
  * `192.168.1.3` (Host 2)
  * `192.168.1.2` (Host 3)
* **Default Gateway:** `192.168.1.1` (Interface `Fa0/0` pada Router0)

### B. Segmen Penghubung Antar-Router (Subnet `192.168.3.0/24`)
* **Router Kiri (Router0) - Interface `Fa0/1`:** `192.168.3.1`
* **Router Kanan (Router1) - Interface `Fa0/0`:** `192.168.3.2`

### C. Segmen Jaringan Lokal Kanan (Subnet `192.168.2.0/24`)
* **Host Tujuan:** `192.168.2.2` (Host 2)
* **Host Lain dalam Subnet:**
  * `192.168.2.3` (Host 1)
* **Default Gateway:** `192.168.2.1` (Interface `Fa0/1` pada Router1)

---

## 2. Pemetaan Tabel Static Route

Untuk menjamin kelancaran komunikasi bolak-balik (dua arah) antar-subnet yang tidak terhubung langsung, kedua router dikonfigurasi dengan entri *static route* sebagai berikut:

| Perangkat Router | Network Target | Subnet Mask | Next-Hop IP | Interface Keluar (Exit Interface) |
| :--- | :--- | :--- | :--- | :--- |
| **Router0 (Kiri)** | `192.168.2.0` | `255.255.255.0` | `192.168.3.2` | `FastEthernet 0/1` |
| **Router1 (Kanan)** | `192.168.1.0` | `255.255.255.0` | `192.168.3.1` | `FastEthernet 0/0` |

---

## 3. Diagram Alur Transmisi Paket (`192.168.1.4` → `192.168.2.2`)

```text
[Host 192.168.1.4]
       │
       ▼ (Evaluasi Subnet Mask & Resolusi ARP Gateway)
   [Switch0]
       │
       ▼
[Router0 - Fa0/0: 192.168.1.1]
       │
       ▼ (Lookup Routing Table: Jalur 192.168.2.0/24 via 192.168.3.2)
[Router0 - Fa0/1: 192.168.3.1]
       │
       ▼ (Transmisi Segmen Inter-Router: Subnet 192.168.3.0/24)
[Router1 - Fa0/0: 192.168.3.2]
       │
       ▼ (Lookup Routing Table: Directly Connected ke 192.168.2.0/24 via Fa0/1)
[Router1 - Fa0/1: 192.168.2.1]
       │
       ▼ (Resolusi ARP Target & Switching Frame)
   [Switch1]
       │
       ▼
[Host 192.168.2.2] (Paket Berhasil Diterima)
```

---

## 4. Rincian Langkah Proses Pengiriman Data

### Tahap 1: Evaluasi Alamat dan Enkapsulasi Awal (Host Sumber `192.168.1.4`)
1. **Analisis Subnet:** Host `192.168.1.4` hendak mentransmisikan data ke IP `192.168.2.2`. Melalui kalkulasi logika *AND* menggunakan netmask `255.255.255.0`, host mendeteksi bahwa alamat sasaran berada di luar subnet lokal (`192.168.1.0/24`).
2. **Pengalihan ke Gateway:** Karena berada di segmen berbeda, paket diarahkan ke *Default Gateway* (`192.168.1.1`).
3. **Resolusi Alamat Fisik (ARP):** Host memeriksa tabel ARP untuk mengambil *MAC Address* interface `Fa0/0` milik Router0 (jika belum tercatat, host mengirimkan siaran *ARP Request* terlebih dahulu).
4. **Penyusunan Frame Layer 2:** Data dienkapsulasi ke dalam Frame Ethernet dengan spesifikasi:
   * **Source IP:** `192.168.1.4`
   * **Destination IP:** `192.168.2.2`
   * **Source MAC:** MAC Address Host `192.168.1.4`
   * **Destination MAC:** MAC Address interface `Fa0/0` Router0

### Tahap 2: Pemrosesan dan Forwarding oleh Router0 (Router Kiri)
1. **Penerimaan & Dekapsulasi:** Frame tiba di interface `Fa0/0`. Router0 melepas lapisan Layer 2 untuk memeriksa header Layer 3 (IP Header).
2. **Pengecekan Routing Table:** Router0 mendeteksi IP tujuan `192.168.2.2`. Berdasarkan entri *Static Route*, jalur ke subnet `192.168.2.0/24` diarahkan menuju *Next-Hop* `192.168.3.2` melalui interface keluar `Fa0/1`.
3. **Pembaruan TTL:** Parameter *Time to Live* (TTL) dikurangi nilainya sebesar 1.
4. **Enkapsulasi Ulang Frame:** Router0 mencari *MAC Address* interface `Fa0/0` Router1 (`192.168.3.2`) melalui cache ARP, lalu membungkus paket ke dalam frame baru:
   * **Source MAC:** MAC Address interface `Fa0/1` Router0
   * **Destination MAC:** MAC Address interface `Fa0/0` Router1
5. **Transmisi:** Frame diluncurkan keluar melewati port `Fa0/1`.

### Tahap 3: Propagasi Antar-Router (Segmen `192.168.3.0/24`)
Frame data melintasi media transmisi segmen penghubung inter-router, bergerak dari `192.168.3.1` (Router0) menuju `192.168.3.2` (Router1).

### Tahap 4: Pemrosesan oleh Router1 (Router Kanan)
1. **Penerimaan:** Router1 menerima frame melalui port `Fa0/0` dan membuka header Layer 2.
2. **Evaluasi Rute Tujuan:** Router1 memeriksa kembali IP tujuan `192.168.2.2` pada tabel routing. Terdeteksi bahwa subnet `192.168.2.0/24` berstatus terhubung langsung (*Directly Connected*) pada port `Fa0/1`.
3. **Pembaruan TTL:** Nilai TTL kembali dikurangi sebesar 1.
4. **Resolusi ARP Host Tujuan:** Router1 mengidentifikasi *MAC Address* dari host target `192.168.2.2` menggunakan tabel ARP lokal (atau mengeksekusi *ARP Request* bila belum tersedia).
5. **Enkapsulasi Akhir:** Paket dibungkus ulang dengan mencantumkan:
   * **Source MAC:** MAC Address interface `Fa0/1` Router1
   * **Destination MAC:** MAC Address Host target `192.168.2.2`
6. **Forwarding:** Frame dilepaskan melalui port `Fa0/1`.

### Tahap 5: Penerimaan Data oleh Host Tujuan (`192.168.2.2`)
1. **Switching:** `Switch1` menerima frame dari port router dan meneruskannya ke port fisik tempat host `192.168.2.2` terhubung.
2. **Verifikasi & Pemrosesan:** Host `192.168.2.2` memvalidasi kesesuaian MAC Address dan IP tujuan. Setelah diverifikasi cocok, enkapsulasi Layer 2 dan Layer 3 dilepaskan, lalu muatan data diteruskan ke layer aplikasi untuk diproses.

---

## 5. Ringkasan Evaluasi

Keberhasilan transmisi paket lintas jaringan non-lokal pada skenario multi-router bergantung pada dua faktor kunci:
1. **Peran Default Gateway:** Memfasilitasi host dalam mengarahkan lalu lintas data ke luar subnet lokal ketika target tidak berada dalam satu ranah broadcast.
2. **Sinkronisasi Static Route:** Router tidak dapat meneruskan lalu lintas menuju subnet yang tidak terhubung langsung tanpa rute manual yang terdefinisi secara konsisten di kedua belah router, memastikan paket data dapat pergi dan pulang dengan rute yang valid.