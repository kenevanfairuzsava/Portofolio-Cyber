
Attack & Defend — Dokumentasi Penetration Testing & Log Analysis

> Proyek kelompok mata kuliah Keamanan Jaringan — Analisis aktivitas attacker dan pembacaan log pada server vulnerable.

---

Deskripsi Proyek

Proyek ini mendokumentasikan simulasi aktivitas attacker terhadap **target server lab** dengan IP `175.17.0.250`, serta analisis log yang dihasilkan menggunakan OSSEC/Wazuh. Tujuan utama adalah memahami bagaimana penyerang melakukan reconnaissance dan exploitation, sekaligus bagaimana defender membaca dan menginterpretasikan log serangan.

---

Target Server

| Service | Port | Versi | Keterangan |
|---------|------|-------|------------|
| FTP | 21 | vsftpd 3.0.5 | Anonymous login, misconfiguration |
| SSH | 22 | OpenSSH 9.6p1 Ubuntu | Brute force target |
| DNS | 53 | — | Domain service |
| HTTP | 80 | Apache 2.4.58 | Web server, TRACE method detected |
| SMB | 139, 445 | Samba 4.19.5 | File sharing enumeration |
| Redis | 6379 | Redis 7.0.15 | Unauthenticated access risk |
| Tomcat | 8080 | Apache Tomcat | Default credential risk |

---

Tahapan Serangan (Attack Chain)

1. Reconnaissance — Nmap

```bash
nmap -sC -sV -p- --min-rate 1000 175.17.0.250
```

| Opsi | Fungsi |
|------|--------|
| `-sC` | Menjalankan NSE default scripts (banner grabbing, enumerasi otomatis) |
| `-sV` | Deteksi versi service (berguna untuk mencari CVE) |
| `-p-` | Full scan semua 65535 port TCP |
| `--min-rate 1000` | Kecepatan minimum 1000 paket/detik |

---

 2. Web Directory Fuzzing — FFUF

```bash
ffuf -w /usr/share/wordlists/dirb/common.txt -u http://175.17.0.250/FUZZ -mc 200,304 -t 10 -c
```

| Opsi | Fungsi |
|------|--------|
| `-w` | Wordlist nama direktori/file umum (~4614 entri) |
| `-u ... /FUZZ` | Placeholder yang diganti tiap entri wordlist |
| `-mc 200,304` | Hanya tampilkan response valid (200/304) |
| `-t 10` | 10 thread paralel |
| `-c` | Output berwarna untuk keterbacaan |

---

 3. SSH Brute Force — Hydra

```bash
hydra -l userserver -P /usr/share/wordlists/rockyou.txt -t 4 -V -f ssh://175.17.0.250
```

| Opsi | Fungsi |
|------|--------|
| `-l userserver` | Single username target |
| `-P rockyou.txt` | Wordlist password legendaris (jutaan entri) |
| `-t 4` | 4 thread (aman untuk SSH, hindari rate limiting) |
| `-V` | Verbose, tampilkan tiap percobaan |
| `-f` | Stop otomatis jika password ditemukan |

---

 4. FTP Brute Force — Hydra

```bash
hydra -L users.txt -P passlist.txt ftp://175.17.0.250 -t 4 -V
```

**Hasil:**
```
[21][ftp] host: 175.17.0.250  login: admin  password: admin
```

Kredensial default `admin:admin` berhasil ditemukan — contoh klasik default credential yang tidak diubah administrator.

---

 5. SMB Enumeration — smbclient

```bash
smbclient //175.17.0.250/public -U anonymous -N
```

| Opsi | Fungsi |
|------|--------|
| `//IP/public` | UNC path menuju share target |
| `-U anonymous` | Login sebagai anonymous user |
| `-N` | No password (null session) |

---

 Analisis Log (Defender Perspective)

 Log SSH

- **Rule ID:** `5710`
- **Tanggal Serangan:** 22 Mei 2026, pukul 03.00
- **IP Penyerang:** `17.17.17.2`
- **User:** `jenkins` (tidak ada di server — invalid user)
- **Teknik:** Password Guessing / Brute Force SSH

---

 Log FTP

- **Rule ID:** `11403` (satu percobaan), `40112` (banyak percobaan + berhasil)
- **Tanggal Serangan:** 25 Mei 2026, pukul 01.00
- **User:** `ftpuser`, **IP:** `17.17.17.2`
- **Teknik:** Brute Force — gagal berulang kali hingga akhirnya berhasil

---

 Log SMB (Samba)

- **Rule ID:** `100200` (satu percobaan), `100210` (banyak percobaan)
- **Tanggal Serangan:** 25 Mei 2026, pukul 01.00
- **Teknik:** Brute Force dengan username yang tidak ada di sistem

---

 Log Apache

- **Rule ID:** `31101` (satu request), `31151` (banyak request)
- **Tanggal Serangan:** 25 Mei 2026, pukul 01.00
- **IP Penyerang:** `17.17.17.2`
- **Teknik:** Vulnerability Scanning terhadap web server Apache

---

 Tools yang Digunakan

| Tool | Fungsi |
|------|--------|
| **Nmap** | Network scanning & service enumeration |
| **FFUF** | Web directory & file fuzzing |
| **Hydra** | Brute force multi-protokol (SSH, FTP, dll) |
| **smbclient** | Akses dan enumerasi share SMB |
| **OSSEC / Wazuh** | Analisis dan pembacaan log serangan |

---

 Kesimpulan

Dokumentasi ini menunjukkan alur penuh penetration testing dari sisi attacker sekaligus cara membaca bukti serangannya dari sisi defender:

1. **Reconnaissance** — Nmap memetakan seluruh attack surface
2. **Enumeration** — FFUF dan smbclient menggali lebih dalam
3. **Exploitation** — Hydra berhasil pada FTP dengan kredensial default `admin:admin`
4. **Log Analysis** — OSSEC/Wazuh merekam setiap aktivitas mencurigakan dengan Rule ID spesifik per protokol

> Pemahaman kedua sisi (attacker & defender) adalah fondasi utama keamanan jaringan yang solid.

---

 Disclaimer

Seluruh aktivitas dalam proyek ini dilakukan di **lingkungan lab yang terkontrol** untuk tujuan **edukasi**. Penggunaan teknik ini di luar lingkungan yang diizinkan adalah ilegal.
