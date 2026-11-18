# ==========================================
# SCRIPT LENGKAP HOTSPOT & L2TP VIA RADIUS
# KHUSUS ROUTEROS v6.47.6
# ==========================================

# 1. KONFIGURASI RADIUS CLIENT (Menghubungkan ke CHR)
# Ganti RAHASIA_RADIUS dengan Secret RADIUS di CHR Anda
/radius
add service=hotspot address=10.10.10.1 secret="RAHASIA_RADIUS" timeout=3s
add service=ppp address=10.10.10.1 secret="RAHASIA_RADIUS" timeout=3s

# ==========================================
# 2. KONFIGURASI HOTSPOT
# ==========================================
# Membuat IP Pool untuk Hotspot
/ip pool
add name=pool_hotspot ranges=192.168.10.50-192.168.10.200

# Membuat Server Profile Hotspot (RADIUS diaktifkan di sini)
/ip hotspot profile
add name=hsprof1 dns-name=login.wifi hotspot=192.168.10.1 login-by=http-chap,http-pap use-radius=yes

# Membuat User Profile Lokal (Sebagai fallback/cadangan)
/ip hotspot user profile
add name=profile-radius

# Menonaktifkan DHCP Server biasa di LAN (Agar tidak bentrok dengan DHCP Hotspot)
/ip dhcp-server disable [find interface=bridge-LAN]

# Mengaktifkan Hotspot di interface LAN
/ip hotspot
add interface=bridge-LAN address-pool=pool_hotspot profile=hsprof1 name=hotspot1

# ==========================================
# 3. KONFIGURASI L2TP SERVER (VPN)
# ==========================================
# Membuat IP Pool untuk Klien VPN
/ip pool
add name=pool-l2tp ranges=192.168.100.10-192.168.100.254

# Membuat PPP Profile untuk L2TP
# Catatan: local-address diubah ke 10.10.10.254 agar tidak bentrok dengan IP CHR (10.10.10.1)
/ppp profile
add name=profile-l2tp local-address=10.10.10.254 remote-address=pool-l2tp use-encryption=yes

# Mengaktifkan L2TP Server dengan IPsec Bawaan (Paling stabil untuk v6)
# Ganti PassKEYmu dengan Pre-Shared Key (PSK) yang Anda inginkan
/interface l2tp-server server
set enabled=yes authentication=pap,chap,mschap2 use-ipsec=yes ipsec-secret="PassKEYmu"

# ==========================================
# 4. FIREWALL (Izinkan Traffic RADIUS & L2TP)
# ==========================================
/ip firewall filter
# Izinkan MikroTik mengirim request RADIUS ke CHR
add action=accept chain=output dst-port=1812,1813 protocol=udp dst-address=10.10.10.1 comment="Allow RADIUS to CHR"

# Izinkan Klien VPN dari luar (WAN) untuk connect L2TP & IPsec
add action=accept chain=input protocol=udp dst-port=500,1701,4500 comment="Allow L2TP/IPsec"
add action=accept chain=input protocol=ipsec-esp comment="Allow IPsec ESP"

# ==========================================
# SELESAI. Konfigurasi bersih dan siap pakai.
# ==========================================

# ==========================================
# KONFIGURASI COA (RADIUS INCOMING)
# ==========================================

# 1. Aktifkan RADIUS Incoming (Mendengarkan request COA/Disconnect dari CHR)
# Port default untuk COA/Disconnect adalah UDP 3799
/radius incoming
set accept=yes port=3799

# 2. Tambahkan Aturan Firewall untuk mengizinkan traffic COA dari CHR
# Pastikan src-address adalah IP CHR Anda (10.10.10.1)
/ip firewall filter
add action=accept chain=input protocol=udp src-address=10.10.10.1 dst-port=3799 comment="Allow RADIUS COA/Disconnect from CHR"

# ==========================================
# SELESAI. COA siap menerima perintah dari CHR.
# ==========================================
