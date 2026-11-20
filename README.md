# Implementasi Jaringan Enterprise MikroTik: Multi-VLAN, L2TP/IPsec Site-to-Site VPN, & Whitelist Firewall

Dokumentasi praktik laboratorium rekayasa jaringan menggunakan **PNETLab** dan **Winbox** untuk merancang infrastruktur jaringan perusahaan yang menghubungkan **Kantor Pusat (HQ)** dan **Kantor Cabang (Branch)**. Proyek ini mengimplementasikan segmentasi jaringan **VLAN (*Router-on-a-Stick*)**, terowongan terenkripsi **L2TP/IPsec VPN**, serta kebijakan keamanan **Firewall Filter berbasis *Whitelist* (*Default Deny*)**.

---

## 1. Arsitektur & Topologi Jaringan

![Topologi Jaringan](<Topologi Jaringan.png>)

### Gambaran Umum Infrastruktur
* **Kantor Pusat (`Router HQ`):** Bertindak sebagai gerbang utama internet, pusat *Inter-VLAN Routing* untuk dua lantai (`Switch Lt1` & `Switch Lt2`), serta bertindak sebagai **L2TP/IPsec VPN Server**.
* **Kantor Cabang (`Router Branch`):** Bertindak sebagai router cabang yang mengelola segmentasi VLAN lokal dan melakukan *dial-out* otomatis sebagai **L2TP/IPsec VPN Client** ke IP Public `Router HQ`.
* **Simulasi ISP (`Router ISP`):** Menyediakan konektivitas IP Public antar-site (`100.100.100.0/24` & `200.200.200.0/24`) serta simulasi target internet (`8.8.8.8/32`).
* **Out-of-Band Management (`Cloud0`):** Jalur terpisah pada `ether4` untuk akses konfigurasi Winbox tanpa mengganggu tabel *routing* utama.

---

## 2. Tabel Pengalamatan IP (IP Addressing Plan)

| Perangkat | Interface | Peran / Segmen Jaringan | IP Address | Keterangan |
| :--- | :--- | :--- | :--- | :--- |
| **Router ISP** | `ether1` | WAN ke `Router HQ` | `100.100.100.1/24` | Gateway ISP HQ |
| **Router ISP** | `ether2` | WAN ke `Router Branch` | `200.200.200.1/24` | Gateway ISP Branch |
| **Router ISP** | `loopback` | Simulasi Target Internet | `8.8.8.8/32` | Bridge Loopback |
| **Router HQ** | `ether1` | WAN Public HQ | `100.100.100.2/24` | Gateway `100.100.100.1` |
| **Router HQ** | `vlan10-staff-hq` | VLAN 10 Staff (Lt1 & Lt2) | `192.168.10.1/24` | DHCP Server (`VPC6`, `VPC8`) |
| **Router HQ** | `vlan20-guest-hq` | VLAN 20 Guest (Lt1 & Lt2) | `192.168.20.1/24` | DHCP Server (`VPC7`, `VPC9`) |
| **Router HQ** | `l2tp-in-branch` | Tunnel VPN Server | `10.10.10.1` | Remote `10.10.10.2` |
| **Router Branch** | `ether1` | WAN Public Branch | `200.200.200.2/24` | Gateway `200.200.200.1` |
| **Router Branch** | `vlan10-staff-br` | VLAN 10 Staff Branch | `192.168.110.1/24` | DHCP Server (`VPC10`) |
| **Router Branch** | `vlan20-guest-br` | VLAN 20 Guest Branch | `192.168.120.1/24` | DHCP Server (`VPC11`) |
| **Router Branch** | `l2tp-to-hq` | Tunnel VPN Client | `10.10.10.2` | Connect To `100.100.100.2` |

---

## 3. Tahapan Konfigurasi Inti

### A. Segmentasi Multi-VLAN (*Router-on-a-Stick* & *Bridge VLAN Filtering*)
* **Kantor Pusat (`Router HQ`):** Menggunakan koneksi Layer 2 murni (*Trunk* tanpa IP fisik) menuju `Switch Lt1` (`ether2`) dan `Switch Lt2` (`ether3`) yang digabungkan dalam satu `bridge-lan` dengan mengaktifkan **VLAN Filtering**, kemudian membuat interface virtual `VLAN 10` dan `VLAN 20`.
* **Switch Layer 2 (`Switch Lt1` & `Switch Lt2`):** Mengonfigurasi port *uplink* ke router sebagai **Tagged (Trunk)** dan port akses menuju klien (`VPC6`–`VPC9`) sebagai **Untagged (Access)** sesuai PVID masing-masing.
* **Kantor Cabang (`Router Branch`):** Mengonfigurasi VLAN 10 (`192.168.110.1/24`) dan VLAN 20 (`192.168.120.1/24`) untuk melayani klien cabang (`VPC10` dan `VPC11`).

![VLAN Bridge Router HQ](<VLAN Bridge Router HQ.png>)

### B. Terowongan Site-to-Site VPN (L2TP/IPsec)
1. **VPN Server (`Router HQ` — `PPP` $\rightarrow$ `L2TP Server` & `Secrets`):**
   * Mengaktifkan **L2TP Server** dengan enkripsi **IPsec (`yes`)** dan *IPsec Secret*.
   * Membuat kredensial akun `vpn-branch` dengan *Tunnel IP* `10.10.10.1` $\leftrightarrow$ `10.10.10.2` serta parameter **Routes:** `192.168.110.0/24, 192.168.120.0/24` agar rute jaringan cabang otomatis ditambahkan saat VPN terhubung.

![VPN Server Router HQ](<VPN Server Router HQ.png>)

2. **VPN Client (`Router Branch` — `PPP` $\rightarrow$ `Interface` $\rightarrow$ `L2TP Client`):**
   * Melakukan koneksi keluar menuju IP Public `100.100.100.2` menggunakan akun `vpn-branch` dan *IPsec Secret* yang sama.
   * Menambahkan rute statis menuju jaringan HQ (`192.168.10.0/24` dan `192.168.20.0/24`) melewati gateway terowongan `10.10.10.1`.

![VPN Client Router Branch](<VPN Client Router HQ.png>)

---

## 4. Kebijakan Keamanan Firewall (*Whitelist / Default Deny*)

Secara bawaan, MikroTik melakukan *Inter-VLAN Routing* secara otomatis antar-subnet yang terhubung. Untuk mencegah kebocoran akses antara jaringan **Guest** dan **Staff**, diterapkan aturan **Firewall Filter berbasis Whitelist** pada chain `forward` di kedua router (mengizinkan hanya lalu lintas yang diperlukan dan memblokir sisanya di baris paling bawah).

### A. Susunan Aturan Firewall `Router Branch`
1. **`#0` (`accept`, `forward`):** Mengizinkan paket balasan (*Connection State:* `established, related`).
2. **`#1` & `#2` (`accept`, `forward`):** Mengizinkan akses internet (`Out. Interface: ether1`) untuk VLAN 10 (`192.168.110.0/24`) dan VLAN 20 (`192.168.120.0/24`).
3. **`#3` & `#5` (`accept`, `forward`):** Mengizinkan komunikasi dua arah sesama **VLAN 10 Staff** (`192.168.110.0/24` $\leftrightarrow$ `192.168.10.0/24`).
4. **`#4` & `#6` (`accept`, `forward`):** Mengizinkan komunikasi dua arah sesama **VLAN 20 Guest** (`192.168.120.0/24` $\leftrightarrow$ `192.168.20.0/24`).
5. **`#7` (`drop`, `forward`):** Menutup seluruh lalu lintas lintas-VLAN lainnya (*Default Deny*).

![Firewall Router Branch](<Firewall Router Branch.png>)

### B. Susunan Aturan Firewall `Router HQ`
1. **`#0` (`drop`, `input`):** Memblokir akses manajemen publik (`TCP 8291, 22, 23`) yang masuk dari interface WAN `ether1`.
2. **`#1` (`accept`, `forward`):** Mengizinkan paket balasan (*Connection State:* `established, related`).
3. **`#2` & `#3` (`accept`, `forward`):** Mengizinkan akses internet (`Out. Interface: ether1`) untuk VLAN 10 (`192.168.10.0/24`) dan VLAN 20 (`192.168.20.0/24`).
4. **`#4` & `#6` (`accept`, `forward`):** Mengizinkan komunikasi dua arah sesama **VLAN 10 Staff** (`192.168.10.0/24` $\leftrightarrow$ `192.168.110.0/24`).
5. **`#5` & `#7` (`accept`, `forward`):** Mengizinkan komunikasi dua arah sesama **VLAN 20 Guest** (`192.168.20.0/24` $\leftrightarrow$ `192.168.120.0/24`).
6. **`#8` (`drop`, `forward`):** Menutup seluruh lalu lintas lintas-VLAN lainnya (*Default Deny*).

![Firewall Router HQ](<Firewall Router HQ.png>)

---

## 5. Matriks Hasil Pengujian & Verifikasi

Pengujian dilakukan menggunakan perintah `ping` dari masing-masing klien (`VPC6` hingga `VPC11`) untuk memvalidasi konektivitas terowongan VPN sekaligus efektivitas isolasi *Firewall Whitelist*:

| Jalur Pengujian (`Sumber` $\rightarrow$ `Tujuan`) | Status Hasil | Analisis Teknis |
| :--- | :--- | :--- |
| **`VPC6` / `VPC8` (Staff HQ)** $\leftrightarrow$ **`VPC10` (Staff Branch)** | **Berhasil (`Reply`)** | Komunikasi sesama VLAN 10 lintas kantor diizinkan melalui terowongan L2TP/IPsec. |
| **`VPC7` / `VPC9` (Guest HQ)** $\leftrightarrow$ **`VPC11` (Guest Branch)** | **Berhasil (`Reply`)** | Komunikasi sesama VLAN 20 lintas kantor diizinkan melalui terowongan L2TP/IPsec. |
| **`VPC7` / `VPC9` (Guest HQ)** $\rightarrow$ **`VPC6` / `VPC8` (Staff HQ)** | **Diblokir (`Timeout`)** | Isolasi antar-VLAN lokal berhasil ditahan oleh aturan `drop` paling bawah di `Router HQ`. |
| **`VPC7` / `VPC9` (Guest HQ)** $\rightarrow$ **`VPC10` (Staff Branch)** | **Diblokir (`Timeout`)** | Guest HQ dicegah mengakses jaringan internal Staff Branch lintas-VPN oleh `Router HQ`. |
| **`VPC11` (Guest Branch)** $\rightarrow$ **`VPC10` (Staff Branch) / `VPC6` (Staff HQ)** | **Diblokir (`Timeout`)** | Guest Branch langsung ditahan di pintu gerbang oleh aturan `drop` pada `Router Branch`. |
| **Seluruh VPC (`VPC6` s.d. `VPC11`)** $\rightarrow$ **`8.8.8.8` (Internet)** | **Berhasil (`Reply`)** | Seluruh segmen jaringan dapat mengakses internet melalui translasi `NAT Masquerade`. |

### Bukti Pengujian Koneksi yang Diizinkan (*Allowed*)
* **`VPC6` (Staff HQ):** Berhasil terhubung ke `VPC10` (Staff Branch) dan internet (`8.8.8.8`).

![Testing Ping Allowed VPC6](<Testing Ping Allowed VPC6.png>)

* **`VPC7` (Guest HQ):** Berhasil terhubung ke `VPC11` (Guest Branch) dan internet (`8.8.8.8`).

![Testing Ping Allowed VPC7](<Testing Ping Allowed VPC7.png>)

### Bukti Pengujian Isolasi Antar-VLAN (*Blocked / Timeout*)
* **`VPC7` (Guest HQ):** Diblokir saat mencoba menjangkau jaringan Staff HQ (`VPC6`) maupun Staff Branch (`VPC10`).

![Testing Ping Blocked VPC7](<Testing Ping Blocked VPC7.png>)

* **`VPC11` (Guest Branch):** Diblokir saat mencoba menjangkau jaringan Staff Branch (`VPC10`) maupun Staff HQ (`VPC6`).

![Testing Ping Blocked VPC11](<Testing Ping Blocked VPC11.png>)
