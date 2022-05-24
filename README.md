# Replikasi MariaDB Master-Slave

Replikasi database berfungsi untuk mereplikasi database dari komputer Master ke Slave yang berbeda, sehingga setiap saat hasil replikasi pada komputer Slave
dapat dibangkitkan menjadi Master jika dibutuhkan. Manfaat lain dari replikasi server dapat berfungsi untuk berbagi beban antara server OLTP dan OLAPs,
dimana server OLTP diharapkan melayani Online Transaksi secara cepat tanpa halangan sedangkan server OLAPs berfungsi untuk pembuatan laporan dan dashboard yang
cenderung memiliki load yang tinggi.

## Persiapan Server Master
Berikut ini adalah langkah-langkah yang dilakukan untuk mempersiapkan Server Master dimana menjadi sumber database yang akan direplikasi ke client.
```
apt-get install mariadb-server mariadb-client -y
pico /etc/mysql/mariadb.conf.d/50-server.cnf
```
dan lakukan setting sebagai berikut:
```
# Instead of skip-networking the default is now to listen only on
# localhost which is more compatible and is not less secure.
bind-address            = 0.0.0.0


# The following can be used as easy to replay backup logs or for replication.
# note: if you are setting up a replication slave, see README.Debian about
#       other settings you may need to change.
server-id               = 1
log_bin                = /var/log/mysql/mysql-bin.log
expire_logs_days        = 10
```
Simpan perubahan tersebut diatas, dan lakukan restart service MariaDB
```
service mariadb restart
```
Lakukan koneksi dengan mysql ke server anda
```
mysql -uroot -p
```
Ketikan perintah berikut ini untuk pembuatan user dan password dan kemampuan REPLICATION SLAVE yang nantinya digunakan oleh server Slave untuk melakukan koneksi untuk
keperluan replikasi.
```
CREATE USER 'replication'@'%' identified by 'your-password';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
FLUSH PRIVILEGES;
```
Selanjutnya adalah menampilkan status dari master
```
show master status;
```
dan akan ditampilkan file log dan posisi yang nantinya akan menjadi acuan oleh Slave untuk keperluan replikasi.
```
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 |      313 |              |                  |
+------------------+----------+--------------+------------------+
```
sampai tahap ini, setting untuk komputer master telah selesai.
## Persiapan Server Slave

