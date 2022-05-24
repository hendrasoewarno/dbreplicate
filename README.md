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

## Mengaktifkan SSL pada Replikasi Master dan Slave
Jika replikasi yang anda lakukan adalah melalui jarigan publik, maka keamanan dari data adalah menjadi konsen yang perlu diperhatikan, sehingga perlu diaktifkan enkripsi data antara Master dan Slave.
### Setting pada Master
Langkah pertama yang perlu dilakukan adalah mempersiapkan sertifikat yang nantinya digunakan pada sisi Master dan sisi slave.
```
$ sudo mkdir -p /etc/mysql/ssl/
$ cd /etc/mysql/ssl/
$ sudo openssl genrsa 2048 > ca-key.pem
## set CA common name to "MariaDB admin" ##
$ sudo openssl req -new -x509 -nodes -days 730 -key ca-key.pem -out ca-cert.pem
## set server certificate common name to "MariaDB master" ##
$ sudo openssl req -newkey rsa:2048 -days 730 -nodes -keyout server-key.pem -out server-req.pem
$ sudo openssl rsa -in server-key.pem -out server-key.pem
$ sudo openssl x509 -req -in server-req.pem -days 730 -CA ca-cert.pem -CAkey ca-key.pem -set_serial 01 -out server-cert.pem
## set client common name to "MariaDB slave" ##
$ sudo openssl req -newkey rsa:2048 -days 730 -nodes -keyout client-key.pem -out client-req.pem
$ sudo openssl rsa -in client-key.pem -out client-key.pem
$ sudo openssl x509 -req -in client-req.pem -days 730 -CA ca-cert.pem -CAkey ca-key.pem -set_serial 01 -out client-cert.pem
$ sudo openssl verify -CAfile ca-cert.pem server-cert.pem client-cert.pem
```
Jika perintah tersebut selesai dilakukan, maka akan menghasilkan file-file berikut ini pada directory /etc/mysql/ssl
```
-rw-r--r-- 1 mysql mysql 1318 May 23 21:53 ca-cert.pem
-rw-r--r-- 1 mysql mysql 1675 May 23 21:52 ca-key.pem
-rw-r--r-- 1 mysql mysql 1172 May 23 21:55 client-cert.pem
-rw------- 1 mysql mysql 1675 May 23 21:55 client-key.pem
-rw-r--r-- 1 mysql mysql  997 May 23 21:55 client-req.pem
-rw-r--r-- 1 mysql mysql 1172 May 23 21:54 server-cert.pem
-rw------- 1 mysql mysql 1679 May 23 21:54 server-key.pem
-rw-r--r-- 1 mysql mysql  997 May 23 21:53 server-req.pem
```
Perlu dipastikan bahwa permission dari directory /etc/mysql/ssl adalah diset owner dan group ke mysql dengan perintah sebagai berikut:
```
chown -R mysql:mysql /etc/mysql/ssl
```
Selanjutnya adalah melakukan setting pada file 50-server.cnf dan 50-client.cnf/
```
pico /etc/mysql/mariadb.conf.d/50-server.cnf
```
dan lakukan perubahan pada file
```
#
# * Security Features
#
# Read the manual, too, if you want chroot!
#chroot = /var/lib/mysql/
#
# For generating SSL certificates you can use for example the GUI tool "tinyca".
#
ssl-ca = /etc/mysql/ssl/ca-cert.pem
ssl-cert = /etc/mysql/ssl/server-cert.pem
ssl-key = /etc/mysql/ssl/server-key.pem
```
```
pico /etc/mysql/mariadb.conf.d/50-client.cnf
```
```
# Example of client certificate usage
ssl-ca=/etc/mysql/ssl/ca-cert.pem
ssl-cert=/etc/mysql/ssl/client-cert.pem
ssl-key=/etc/mysql/ssl/client-key.pem
```
Setelah melakukan setting tersebut diatas, perlu direstart service MariaDB
```
service mariadb restart
```
Selanjutnya adalah melakukan pemeriksaan hasil pengaktifan SSL pada server dengan keberadaan baris SSL: Cipher in use is DHE-RSA-AES256-SHA
```
mysql -uroot -p

status;

--------------
mysql  Ver 15.1 Distrib 10.3.24-MariaDB, for debian-linux-gnu (x86_64) using readline 5.2

Connection id:          50
Current database:
Current user:           root@localhost
SSL:                    Cipher in use is DHE-RSA-AES256-SHA
Current pager:          stdout
Using outfile:          ''
Using delimiter:        ;
Server:                 MariaDB
Server version:         10.3.24-MariaDB-2-log Debian buildd-unstable
Protocol version:       10
Connection:             Localhost via UNIX socket
Server characterset:    utf8mb4
Db     characterset:    utf8mb4
Client characterset:    utf8mb4
Conn.  characterset:    utf8mb4
UNIX socket:            /var/run/mysqld/mysqld.sock
Uptime:                 1 hour 49 min 42 sec

Threads: 9  Questions: 128  Slow queries: 2  Opens: 33  Flush tables: 1  Open tables: 27  Queries per second avg: 0.019
--------------
```
### Setting pada Slave
Duplikasi semua file sertifikat yang ada pada Master (/etc/mysql/ssl) ke komputer Slave pada directory yang sama, dan lakukan perubahan pada 50-client.cnf
pico /etc/mysql/mariadb.conf.d/50-client.cnf
```
```
# Example of client certificate usage
ssl-ca=/etc/mysql/ssl/ca-cert.pem
ssl-cert=/etc/mysql/ssl/client-cert.pem
ssl-key=/etc/mysql/ssl/client-key.pem
```
Setelah melakukan setting tersebut diatas, perlu direstart service MariaDB
