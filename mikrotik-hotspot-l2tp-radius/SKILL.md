name: mikrotik-hotspot-l2tp-radius
description: Konfigurasi lengkap Hotspot + L2TP VPN via RADIUS client + COA/Disconnect di RouterOS v6.47.6, sudah teruji dan bekerja.
version: 1.1.0
triggers:
  - "setup hotspot radius"
  - "konfigurasi l2tp radius"
  - "hotspot l2tp mikrotik"
  - "radius client mikrotik"
  - "script hotspot l2tp"
  - "konfigurasi coa radius"
  - "radius incoming mikrotik"
  - "coa disconnect mikrotik"
tools_required:
  - terminal
---

# Konfigurasi Hotspot + L2TP via RADIUS + COA (RouterOS v6.47.6)

## Tujuan
Menerapkan konfigurasi lengkap pada router MikroTik yang berjalan di **RouterOS v6.47.6** untuk:
1. Menjadikan router sebagai **RADIUS client** ke server CHR di `10.10.10.1`.
2. Mengaktifkan **Hotspot** di `bridge-LAN` dengan autentikasi via RADIUS.
3. Mengaktifkan **L2TP Server** dengan IPsec (PSK) dan autentikasi via RADIUS.
4. Mengaktifkan **RADIUS Incoming (COA/Disconnect)** di port UDP 3799 agar CHR bisa memutus sesi atau mengubah profil pengguna secara real-time.
5. Menambahkan aturan firewall yang diperlukan.

> **PENTING**: Script ini HANYA untuk RouterOS v6.x (diuji di v6.47.6). Jangan terapkan ke v7 tanpa penyesuaian (sintaks `/interface l2tp-server server` berbeda di v7).

## Variabel yang Harus Dikonfirmasi Sebelum Eksekusi
Tanyakan kepada pengguna dan ganti placeholder berikut:
- `RAHASIA_RADIUS` → secret RADIUS yang sama dengan di CHR.
- `L4ll14n4` → Pre-Shared Key IPsec yang diinginkan.
- Alamat CHR: default `10.10.10.1` (konfirmasi jika berbeda).
- Interface LAN: default `bridge-LAN` (konfirmasi nama interface sebenarnya).
- Port COA: default `3799` (konfirmasi jika CHR dikonfigurasi berbeda).

## Langkah Eksekusi

### 1. Konfigurasi RADIUS Client
Jalankan via SSH ke router:
```
/radius
add service=hotspot address=10.10.10.1 secret="RAHASIA_RADIUS" timeout=3s
add service=ppp address=10.10.10.1 secret="RAHASIA_RADIUS" timeout=3s
```

### 2. Konfigurasi Hotspot
```
/ip pool
add name=pool_hotspot ranges=192.168.10.50-192.168.10.200

/ip hotspot profile
add name=hsprof1 dns-name=login.wifi hotspot=192.168.10.1 login-by=http-chap,http-pap use-radius=yes

/ip hotspot user profile
add name=profile-radius

/ip dhcp-server disable [find interface=bridge-LAN]

/ip hotspot
add interface=bridge-LAN address-pool=pool_hotspot profile=hsprof1 name=hotspot1
```

### 3. Konfigurasi L2TP Server
```
/ip pool
add name=pool-l2tp ranges=192.168.100.10-192.168.100.254

/ppp profile
add name=profile-l2tp local-address=10.10.10.254 remote-address=pool-l2tp use-encryption=yes

/interface l2tp-server server
set enabled=yes authentication=pap,chap,mschap2 use-ipsec=yes ipsec-secret="L4ll14n4"
```
**Catatan penting**: `local-address=10.10.10.254` sengaja dipilih agar tidak bentrok dengan IP CHR (`10.10.10.1`). Jangan diubah ke `10.10.10.1`.

### 4. Konfigurasi RADIUS Incoming (COA/Disconnect)
Aktifkan listener COA/Disconnect agar CHR dapat mengirim perintah `CoA-Request` atau `Disconnect-Request` ke router:
```
/radius incoming
set accept=yes port=3799
```
**Penjelasan**: Port `3799` adalah standar RFC 5176 untuk Dynamic Authorization (COA/Disconnect). Setelah ini aktif, CHR dapat memutus sesi hotspot/PPP secara paksa atau mengubah rate-limit tanpa menunggu user logout.

### 5. Aturan Firewall
```
/ip firewall filter
add action=accept chain=output dst-port=1812,1813 protocol=udp dst-address=10.10.10.1 comment="Allow RADIUS to CHR"
add action=accept chain=input protocol=udp dst-port=500,1701,4500 comment="Allow L2TP/IPsec"
add action=accept chain=input protocol=ipsec-esp comment="Allow IPsec ESP"
add action=accept chain=input protocol=udp src-address=10.10.10.1 dst-port=3799 comment="Allow RADIUS COA/Disconnect from CHR"
```
**Catatan keamanan**: Aturan COA sengaja dibatasi `src-address=10.10.10.1` (hanya CHR) agar pihak lain tidak bisa mengirim paket Disconnect palsu untuk memutus user Anda.

## Verifikasi Setelah Eksekusi
Jalankan perintah berikut dan laporkan hasilnya:
- `/radius print` → pastikan 2 entri aktif (hotspot & ppp).
- `/radius incoming print` → pastikan `accept=yes` dan `port=3799`.
- `/ip hotspot print` → pastikan `hotspot1` berjalan di `bridge-LAN`.
- `/interface l2tp-server server print` → pastikan `enabled=yes`.
- `/ip firewall filter print` → pastikan 4 aturan baru ada.
- `/log print where topics~"radius"` → cek apakah ada error koneksi ke CHR.

## Uji Coba COA (Opsional, Jika Diminta Pengguna)
Jika pengguna ingin memverifikasi COA benar-benar berfungsi, minta mereka menjalankan dari sisi CHR:
```
echo "User-Name=testuser" | radclient -x 10.10.10.2:3799 disconnect "RAHASIA_RADIUS"
```
(Ganti `10.10.10.2` dengan IP MikroTik yang terlihat dari CHR, dan `testuser` dengan user hotspot/PPP yang sedang aktif.)
Jika berhasil, sesi user akan langsung terputus, dan log MikroTik akan menampilkan pesan disconnect.

## Catatan & Peringatan
- **Backup dulu**: sebelum eksekusi, sarankan pengguna menjalankan `/system backup save name=before-radius` dan `/export file=before-radius`.
- **DHCP Server dinonaktifkan**: DHCP server di `bridge-LAN` akan dimatikan karena DHCP Hotspot yang akan menangani. Pastikan ini disetujui pengguna.
- **Port 1701 di firewall**: hanya diperlukan jika ada klien L2TP tanpa IPsec (pure L2TP). Jika semua klien pakai IPsec, port 1701 bisa dihilangkan.
- **Jangan jalankan dua kali**: perintah `add` akan gagal jika entri dengan nama sama sudah ada. Sebelum eksekusi, cek dulu dengan `print` untuk menghindari duplikasi.
- **COA butuh secret yang sama**: pastikan secret RADIUS di CHR dan di MikroTik identik, karena paket COA/Disconnect juga diverifikasi dengan secret tersebut.
- **Hanya baca dulu**: jika pengguna ingin aman, jalankan semua perintah `print` terlebih dahulu untuk memetakan kondisi awal, baru kemudian eksekusi `add`/`set` setelah dikonfirmasi.