# Replikasi MariaDB Master-Slave

Replikasi database berfungsi untuk mereplikasi database dari komputer Master ke Slave yang berbeda, sehingga setiap saat hasil replikasi pada komputer Slave
dapat dibangkitkan menjadi Master jika dibutuhkan. Manfaat lain dari replikasi server dapat berfungsi untuk berbagi beban antara server OLTP dan OLAPs,
dimana server OLTP diharapkan melayani Online Transaksi secara cepat tanpa halangan sedangkan server OLAPs berfungsi untuk pembuatan laporan dan dashboard yang cenderung memiliki load yang tinggi.

Nb. Kami melakukan di MariaDB 10.3

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
Lakukan instalasi komputer klien sebagaimana komputer server diatas, dan lakukan perubahan setting sebagai berikut:
```
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
server-id               = 2
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
Ketikan perintah berikut untuk menghentikan slave, setting koneksi ke Master, terkait file log dan posisi.
```
stop slave;
CHANGE MASTER TO MASTER_HOST = 'your-master-host-ip', MASTER_USER = 'replication', MASTER_PASSWORD = 'your-password', MASTER_LOG_FILE = 'mysql-bin.000001', MASTER_LOG_POS = 313;
start slave
```
Ketikan perintah berikut ini untuk melihat status replikasi
```
show slave status\G;
```
Pastikan tidak error, terutama koneksi dari slave ke master, dan terdapat info Slave has read all relay log; waiting for the slave I/O thread to update it.
```
*************************** 1. row ***************************
                Slave_IO_State: Waiting for master to send event
                   Master_Host: 172.21.5.216
                   Master_User: replication
                   Master_Port: 3306
                 Connect_Retry: 60
               Master_Log_File: mysql-bin.000020
           Read_Master_Log_Pos: 688
                Relay_Log_File: mysqld-relay-bin.000002
                 Relay_Log_Pos: 901
         Relay_Master_Log_File: mysql-bin.000020
              Slave_IO_Running: Yes
...
       Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
```
## Menguji keberhasilan replikasi Master Slave
Untuk menguji keberhasilan replikasi, maka anda dapat mencoba menambahkan database/tabel/data pada master, dan memeriksa kembali ke klien apakah tereplikasi
ke klien.
