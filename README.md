Jawaban UTS Keamanan Jaringan - Kasus 5
Nama: Danny Ahmad Gunawan
NIM: 24552011168 (Jumlah digit: 35, sisa bagi 5: 0 -> Kasus 5)
Direktori Proyek: C:\uts-netsec-k5

Persiapan Lab - Docker Compose & Konfigurasi Awal

Penjelasan Setiap Tahapan

Pembuatan direktori proyek: Membuat folder C:\uts-netsec-k5 di Command Prompt Windows sebagai pusat penyimpanan docker-compose.yml, skrip praktik, dan berkas laporan.

Penyusunan docker-compose.yml: Mendefinisikan jaringan internal terisolasi uts_k5_net (driver: bridge, internal: true, subnet 172.25.5.0/24), volume vol-k5 dan lab-k5-vol, serta empat container:

target-k5 (nginx:alpine, IP 172.25.5.10, port 80/tcp, mount vol-k5 read-only)

gateway-k5 (alpine:latest, IP 172.25.5.254, NET_ADMIN)

korban-k5 (alpine:latest, IP 172.25.5.20, default route ke 172.25.5.254, NET_ADMIN & NET_RAW)

penyerang-k5 (kalilinux/kali-rolling, IP 172.25.5.100, mount lab-k5-vol:/lab-k5, NET_ADMIN & NET_RAW)

Perintah / Kode

mkdir C:\uts-netsec-k5
cd /d C:\uts-netsec-k5
notepad C:\uts-netsec-k5\docker-compose.yml


Bukti Pendukung


Pertemuan 1 (Tugas 1) - Docker, Volume, dan Compose

Penjelasan Setiap Tahapan

Menjalankan stack compose: Menjalankan docker compose up -d --force-recreate dan docker compose ps untuk mengaktifkan seluruh container, membuat volume dan jaringan internal, serta memverifikasi port 80/tcp pada target-k5.

Menguji persistensi volume: Menulis NIM 24552011168 ke /data/laporan-k5.txt di dalam volume vol-k5 dari container sementara (--rm), menghapus container tersebut, lalu membaca ulang file dari container baru yang memasang volume yang sama (vol-k5). Hasilnya tetap 24552011168.

Akses HTTP & inspeksi jaringan: Mengakses layanan HTTP target-k5 (172.25.5.10:80) dari penyerang-k5 serta memeriksa docker network inspect uts-netsec-k5_uts_k5_net ("Internal": true, subnet 172.25.5.0/24). Respons yang tampil adalah HTTP/1.1 200 OK (Server: nginx/1.31.10) beserta halaman Welcome to nginx!.

Perintah / Kode

docker compose up -d --force-recreate
docker compose ps
docker run --rm -v vol-k5:/data alpine sh -c "echo '24552011168' > /data/laporan-k5.txt"
docker run --rm -v vol-k5:/data alpine cat /data/laporan-k5.txt
docker exec penyerang-k5 bash -c "exec 3<>/dev/tcp/172.25.5.10/80 && printf 'GET / HTTP/1.0\r\nHost: 172.25.5.10\r\n\r\n' >&3 && cat <&3"
docker network inspect uts-netsec-k5_uts_k5_net


Bukti Pendukung

image_2f0e1a.png, image_2f1148.png, image_2f1180.png, image_2f11a5.png, image_2f11fd.png, image_2f1528.png, image_2f1580.png, image_2f159e.png, image_2f1868.png, image_2f18a6.png

Pertemuan 2 (Tugas 2) - Kriptografi

Penjelasan Setiap Tahapan

Enkripsi, dekripsi, dan verifikasi SHA-256: Skrip p2.sh menulis 24552011168 ke laporan-k5.txt, menghitung hash SHA-256 sebelum enkripsi, mengenkripsi data menjadi laporan-k5.enc.hex, mendekripsinya kembali menjadi laporan-k5.dec.txt, dan membandingkan hash SHA-256 sesudah dekripsi.

Pembersihan karakter CRLF: Karena file yang diedit di Notepad Windows memiliki karakter \r, skrip dibersihkan dengan tr -d '\r' menjadi /lab-k5/p2_clean.sh lalu dijalankan menggunakan sh.

Sertifikat self-signed: Membuat sertifikat lab k5-exfil.crt dengan CN=k5.exfil.lab serta menampilkan Subject dan Issuer-nya. Hasilnya: SHA-256 sebelum dan sesudah dekripsi identik (4d35d7c62f4957e48f626e67fe4bfe98c2870badf78a58f5fc36c8c8ca986a8b), hasil dekripsi kembali menjadi 24552011168, serta Subject: CN=k5.exfil.lab dan Issuer: CN=k5.exfil.lab.

Perintah / Kode

notepad C:\uts-netsec-k5\p2.sh
docker cp C:\uts-netsec-k5\p2.sh penyerang-k5:/lab-k5/p2.sh
docker exec penyerang-k5 sh -c "tr -d '\r' < /lab-k5/p2.sh > /lab-k5/p2_clean.sh && sh /lab-k5/p2_clean.sh"


Jawaban Konsep

Perbedaan sertifikat self-signed dan CA: Sertifikat self-signed ditandatangani langsung menggunakan kunci privat entitas itu sendiri sehingga tidak otomatis dipercaya klien lain tanpa pemasangan sertifikat secara manual. Sebaliknya, sertifikat yang ditandatangani Certificate Authority (CA) diterbitkan oleh otoritas tepercaya yang sertifikat root/intermediate-nya sudah tertanam pada sistem operasi atau browser sehingga identitas server dapat diverifikasi secara terpusat.

Bukti Pendukung

image_2f827e.png, image_2f8584.png, image_2f85c0.png, image_2f9084.png

Pertemuan 3 (Tugas 3) - ARP Spoofing

Penjelasan Setiap Tahapan

Pencatatan tabel ARP awal: Pada jaringan 172.25.5.0/24, tabel /proc/net/arp pada korban-k5 (172.25.5.20) diperiksa dan mencatat gateway 172.25.5.254 berpasangan dengan MAC address ca:63:2c:fb:64:61 (Flags 0x2, eth0).

Skenario spoofing & pemulihan static ARP: Saat ARP spoofing berlangsung, penyerang mengirim balasan ARP palsu agar IP gateway 172.25.5.254 di tabel ARP korban mengarah ke MAC penyerang (aa:26:f1:62:15:4b), lalu dipulihkan kembali ke MAC gateway yang benar (ca:63:2c:fb:64:61) melalui entri static ARP / permanen.

Perintah / Kode

docker exec korban-k5 sh -c "ping -c 1 172.25.5.254 >/dev/null 2>&1; echo '=== ARP ==='; cat /proc/net/arp"


Jawaban Konsep

Mitigasi selain static ARP: Dynamic ARP Inspection (DAI) pada switch terkelola, yang memvalidasi paket ARP masuk terhadap tabel ikatan IP-MAC-Port (DHCP snooping binding table) dan membuang paket ARP yang tidak sah.

Bukti Pendukung

image_2f90e7.png

Pertemuan 4 (Tugas 4) - Analisis Handshake WPA2

Penjelasan Setiap Tahapan

Analisis file capture & wordlist: Menjalankan aircrack-ng dengan file capture 4-way handshake WPA2 dan wordlist yang disediakan untuk menguji Pairwise Master Key (PMK) berdasarkan passphrase dan SSID.

Interpretasi hasil: Mencatat passphrase yang ditemukan jika cocok pada wordlist, atau menampilkan KEY NOT FOUND apabila kata sandi target tidak ada di dalam wordlist.

Perintah / Kode

aircrack-ng -w <wordlist.txt> -b <BSSID_AP> <capture_file.cap>


Jawaban Konsep

Empat langkah 4-way handshake WPA2:

Pesan 1 (AP -> Klien): Access Point (AP) mengirim ANonce kepada klien.

Pesan 2 (Klien -> AP): Klien menurunkan Pairwise Transient Key (PTK) lalu mengirim SNonce dan Message Integrity Code (MIC) ke AP.

Pesan 3 (AP -> Klien): AP memverifikasi MIC, menurunkan PTK yang sama, dan mengirim Group Temporal Key (GTK) terenkripsi beserta MIC serta instruksi pemasangan kunci.

Pesan 4 (Klien -> AP): Klien mengirim konfirmasi pemasangan kunci (ACK) dan mengaktifkan enkripsi sesi.

Bukti Pendukung

image_2f919c.png

Pertemuan 5 (Tugas 5) - ICMP dengan Scapy

Penjelasan Setiap Tahapan

Pengiriman & pembacaan balasan ICMP: Mengirim ICMP Echo Request ke target (172.25.5.10) dan membaca respons ICMP Echo Reply, terverifikasi pula melalui uji ICMP ping -c 2 172.25.5.10 dari korban-k5.

Verifikasi parameter paket: Hasil balasan ICMP menunjukkan 0% packet loss, TTL = 64, dan balasan ICMP Echo Reply (Type = 0).

Perintah / Kode

docker exec korban-k5 sh -c "echo '=== PING TARGET ==='; ping -c 2 172.25.5.10"


Jawaban Konsep

Penjelasan smurf attack: Smurf attack adalah serangan Denial of Service (DoS) berbasis ICMP di mana penyerang mengirim paket ICMP Echo Request dengan alamat IP sumber palsu (spoofed menjadi alamat IP korban) ke alamat broadcast jaringan sehingga banyak host membalas Echo Reply ke korban secara bersamaan dan membanjiri sumber daya korban.

Bukti Pendukung

image_2f90e7.png

Pertemuan 6 (Tugas 6) - SYN Flood dan Three-Way Handshake

Penjelasan Setiap Tahapan

Pemeriksaan soket TCP & jumlah SYN_RECV: Memeriksa /proc/net/tcp pada target-k5 di mana port 0050 (heksadesimal untuk port 80) berstatus 0A (TCP_LISTEN), serta mencatat jumlah soket SYN_RECV (kode state 03) sebelum serangan dan saat SYN flood berlangsung selama 15 detik.

Three-way handshake & koneksi HTTP: Menangkap paket saat satu koneksi HTTP dibuat (SYN -> SYN-ACK -> ACK sebelum permintaan HTTP dan balasan HTTP/1.1 200 OK dari nginx/1.31.10).

Perintah / Kode

docker exec target-k5 sh -c "echo '=== /proc/net/tcp ==='; cat /proc/net/tcp"
docker exec penyerang-k5 bash -c "exec 3<>/dev/tcp/172.25.5.10/80 && printf 'GET / HTTP/1.0\r\nHost: 172.25.5.10\r\n\r\n' >&3 && head -n 5 <&3"


Jawaban Konsep

Satu mitigasi SYN flood: TCP SYN cookies, yaitu menyandikan informasi sesi ke dalam Initial Sequence Number (ISN) paket SYN-ACK tanpa mengalokasikan entri backlog penuh sebelum paket ACK yang valid diterima.

Bukti Pendukung

image_2f9124.png

Pertemuan 7 (Tugas 7) - DNS Exfiltration

Penjelasan Setiap Tahapan

Encoding NIM ke hex & pengiriman subdomain: Mengubah NIM 24552011168 menjadi representasi heksadesimal ASCII 3234353532303131313638, lalu mengirim label hex tersebut sebagai subdomain ke server DNS lab.

Analisis log & decoding kembali ke NIM: Menganalisis log DNS dengan skrip deteksi untuk mengidentifikasi label hex yang terdeteksi, lalu men-decode kembali nilai hex menjadi NIM 24552011168.

Perintah / Kode

docker exec penyerang-k5 bash -c "printf '24552011168' | od -An -tx1 | tr -d ' \n\r'"
docker exec penyerang-k5 bash -c "printf '\x32\x34\x35\x35\x32\x30\x31\x31\x31\x36\x38\n'"


Jawaban Konsep

Mengapa sulit diblokir firewall biasa: Memanfaatkan protokol/port DNS (UDP/TCP port 53) yang umumnya diizinkan keluar jaringan agar resolusi nama domain tetap berjalan.

Satu indikator deteksi: Panjang subdomain/label yang tidak wajar, entropi karakter tinggi (acak/encoded), serta lonjakan volume query ke domain tertentu.

Bukti Pendukung

image_2f915f.png
