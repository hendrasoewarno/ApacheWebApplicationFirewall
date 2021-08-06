## INSTALASI MOD-SECURITY (Web Application Firewall) pada Apache2
Upaya pengembangan WEB membutuhkan startegi pengamanan yang berlapis, yaitu:
1. Menggunakan sistem operasi dengan setting dan konfigurasi yang aman
2. Menggunakan webserver, scripting dan database server dengan setting dan konfigurasi yang aman
3. Melakukan pemeriksaan terhadap semua POST/GET sebelum diproses untuk menghindari potensi serangan SQL Injection, XSS, maupun upaya melanggar hak akses
4. Memasang Web Application Firewall untuk pertahanan berlapis atas segala response dan request.
Pada tulisan ini akan membahas terkait dengan instalasi dan setting module MOD-Security pada Apache2.
```
curl -i localhost
```
Perhatikan hasil response pada baris Server: ...
```
HTTP/1.1 200 OK
Date: Tue, 28 Apr 2009 22:06:21 GMT
Server: Apache/2.2.11 (Ubuntu) PHP/5.2.6-3ubuntu4.1 with Suhosin-Patch
Last-Modified: Tue, 28 Apr 2009 21:39:54 GMT
ETag: “50d4a-2d-468a44dadbe80”
Accept-Ranges: bytes
Content-Length: 45
Vary: Accept-Encoding
Content-Type: text/html
```
Proses instalasi
```
apt-get install libapache-mod-security
```
atau untuk versi ubuntu 14.02 keatas
```
apt-get install libapache-mod-security2
```
dan aktifkan mod-security pada apache2
```
a2enmod mod-security
service apache2 restart
```
Periksa module mod_security telah diaktifkan dengan perintah apache2ctl -M
```
apache2ctl -M
```
dan periksa apakah ada baris security2_module (shared), kemudian buatlah file konfigurasikan untuk kebutuhan memasukan rules mod-security
```
echo "Include /etc/apache2/modsecurity/*.conf" > /etc/apache2/conf.d/modsecurity2.conf
```
kemudian membuat membuat symbolic link untuk mengalihkan semua log file mod_security dari /etc/apache2/logs ke /var/log/apache2/mod_security
```
mkdir /var/log/apache2/mod_security
ln -s /var/log/apache2/mod_security/ /etc/apache2/logs
```
dan selanjutnya membuat folder /etc/apache2/modsecurity untuk menampung rules mod-security
```
mkdir /etc/apache2/modsecurity
```
Selanjutkan adalah mempersiapkan rules mod-security yang terdapat pada file modsecurity-core-rules_2.5-1.6.1.tar.gz, download dan copy ke folder /etc/apache2/modsecurity, ekstrak ke folder yang sama (tidak membuat sub folder), dan hapus file yang tidak digunakan.
```
cd /etc/apache2/modsecurity
tar xzvf modsecurity-core-rules_2.5-1.6.1.tar.gz
rm CHANGELOG LICENSE README modsecurity-core-rules_2.5-1.6.1.tar.gz
```
Restart kembali service Apache2
```
service apache2 restart
```
### Periksa Hasil instalasi
Jalankan perintah berikut untuk melihat response header setelah aplikasi dari WAF
```
curl -i localhost
```
Perhatikan hasil response pada baris Server: ...
```
HTTP/1.1 200 OK
Date: Tue, 28 Apr 2009 22:06:21 GMT
Server: Apache/2.2.11 (Ubuntu)
Last-Modified: Tue, 28 Apr 2009 21:39:54 GMT
ETag: “50d4a-2d-468a44dadbe80”
Accept-Ranges: bytes
Content-Length: 45
Vary: Accept-Encoding
Content-Type: text/html
```
Buka file /etc/apache2/conf.d/security, dan ubah:
```
ServerTokens Prod
ServerSignature Off
TraceEnable Off
```
Restart Apache2
```
service apache2 restart
curl -i localhost
```
Perhatikan kembali hasil response pada baris Server: ...
```
HTTP/1.1 200 OK
Date: Tue, 28 Apr 2009 22:06:21 GMT
Server: Apache
Last-Modified: Tue, 28 Apr 2009 21:39:54 GMT
ETag: “50d4a-2d-468a44dadbe80”
Accept-Ranges: bytes
Content-Length: 45
Vary: Accept-Encoding
Content-Type: text/html
```
Jika anda juga ingin menyamarkan informasi Server: Apache menjadi NodeJS (walaupun belum tentu efektif), maka pada Mod-Security dapat dilakukan dengan menambahkan baris:
```
pico /etc/apache2/cond.d/modsecurity/modsecurity_crs_10_config.conf
	SecServerSignature "NodeJS"
```
dan hasil response header akan menjadi:
```
HTTP/1.1 200 OK
Date: Tue, 28 Apr 2009 22:06:21 GMT
Server: NodeJS
Last-Modified: Tue, 28 Apr 2009 21:39:54 GMT
ETag: “50d4a-2d-468a44dadbe80”
Accept-Ranges: bytes
Content-Length: 45
Vary: Accept-Encoding
Content-Type: text/html
```
Awalnya Mod-Security akan bekerja pada modus Error Log saja, dan meneruskan request, jika anda ingin menolak request yang terdeteksi oleh rule, maka buka file /etc/apache2/cond.d/modsecurity/modsecurity_crs_10_config.conf dan hilangkan # pada SecDefaultAction ...
```
pico /etc/apache2/cond.d/modsecurity/modsecurity_crs_10_config.conf
	SecDefaultAction "phase:2,log,deny,status:403,t:lowercase,t:replaceNulls,t:compressWhitespace"
```
Log dari mod_security dapat dibaca di /var/log/apache2/mod_security yang terdiri dari file modsec_audit.log.
Selanjutnya adalah anda perlu melakukan review dengan mengaktifkan ataupun mematikan rules yang terdapat pada folder /etc/apache2/conf.d/modsecurity untuk mengefektifkan rule dan meminimalkan impact kepada aplikasi anda
### Non-Aktifkan Mod-Security
jika setelah pengaktifan rules terdapat anomali pada aplikasi anda, maka anda dapat mengubah modus dari mod_security ke level DetectionOnly (dari On), dengan melakukan perubahan pada file /etc/apache2/cond.d/modsecurity/modsecurity_crs_10_config.conf.
```
pico /etc/apache2/cond.d/modsecurity/modsecurity_crs_10_config.conf
```
dan ubah setting ke
```
SecRuleEngine DetectionOnly
```
### Non-Aktifkan Mod-Security terhadap Response ke User
Jika setelah implementasi ditemukan banyak halaman WEB anda yang awalnya berjalan dengan baik, tetapi sekarang mendapatkan pesan <b>Internal Server Error</b>, hal ini berarti bahwa hasil pemeriksaan response dari halaman WEB anda ke client juga mengandung script yang beresiko. Jika anda menjalankan Mod-Security hanya untuk mendeteksi request dari user dan mengabaikan response ke user, maka anda dapat mempertimbangkan untuk mengubah setting pada modsecurity_crs_10_config.conf menjadi:
```
SecResponseBodyAccess Off
```
### Non-Aktifkan Mod-Security terhadap aplikasi tertentu
Jika setelah implementasi ditemukan beberapa aplikasi yang awalnya berjalan dengan baik, tetapi sekarang mendapatkan pesan <b>Internal Server Error</b>, sehingga anda ingin Mod-Security mengabaikan aplikasi tertentu, maka anda dapat menambahkan SecRuleEngine Off pada setting directory aplikasi anda.
```
<Directory /var/www/site/phpMyAdmin>
SecRuleEngine Off
</Directory>
```
### Non-Aktifkan Mod-Security terhadap URI tertentu
Jika dalam implementasi anda ingin Mod-Security mengabaikan URI tertentu, maka anda dapat menggunakan setting sebagai berikut:

```
SecRule REQUEST_URI "@beginsWith /aaaa/bbbb.php" "id:1,phase:2,pass,nolog,allow,ctl:ruleEngine=Off"
```
Dengan catatan Phase(1) adalah mengabaikan request Header, Phase(2) adalah mengabaikan request Body, Phase(3) adalah mengabaikan response Header, dan Phase(4) adalah mengabaikan response Body.

### Non-Aktifkan Rule Mod-Security berdasarkan Id
Jika anda menemukan ada beberapa rule yang tidak sesuai dengan prilaku aplikasi, dan anda ingin mengabaikan rule tersebut berdasarkan id:
```
<IfModule mod_security2.c>
    SecRuleRemoveById 950109 950901 958291 dst
</IfModule>
```
Pendekatan lain untuk menon-aktifkan rule adalah melakukan remark langsung pada file conf rule terkait.
### Modifikasi Rule
Misalkan anda ingin menambahkan prilaku rule tertentu agar dapat mengenali tanda remark (-- dan #) yang banyak dipakai pada SQL Injection untuk mendisable operasi berikutnya. pada file modsecurity_crs_40_generic_attacks.conf, baris 71 atau rule id 959901:
```
SecRule REQUEST_HEADERS|XML:/*|!REQUEST_HEADERS:Referer "\b(\d+) ?= ?\1\b|[\'\"](\w+)[\'\"] ?= ?[\'\"]\2\b" \
        "phase:2,capture,t:none,t:urlDecodeUni,t:htmlEntityDecode,t:replaceComments,t:compressWhiteSpace,t:lowercase,ctl:auditLogParts=+E,deny,log,auditlog,status:501,msg:'SQL Injection Attack',id:'959901',tag:'WEB_ATTACK/SQL_INJECTION',logdata:'%{TX.0}',severity:'2'"
```
menjadi
```
SecRule REQUEST_HEADERS|XML:/*|!REQUEST_HEADERS:Referer "\b(\d+) ?= ?\1\b|[\'\"](\w+)[\'\"] ?= ?[\'\"]\2\b|.*[-]{2,}.*|.*#.*" \
        "phase:2,capture,t:none,t:urlDecodeUni,t:htmlEntityDecode,t:replaceComments,t:compressWhiteSpace,t:lowercase,ctl:auditLogParts=+E,deny,log,auditlog,status:501,msg:'SQL Injection Attack',id:'959901',tag:'WEB_ATTACK/SQL_INJECTION',logdata:'%{TX.0}',severity:'2'"
```
Ingat, pada setiap perubahan setting, maka jangan lupa melakukan restart server Apache anda.
### Mengatur logrotate
Untuk mengatur logrotate anda perlu menambahkan script berikut ini pada /etc/logrotate
```
/var/log/apache2/mod_security/modsec_audit.log {
    rotate 7
    compress
    daily
    postrotate
       /etc/init.d/apache2 reload > /dev/null 2>&1 || true
    endscript
}
```
Jika anda tidak menambahkan postrotate untuk apache2 reload akan menyebabkan MOD-Security tidak akan sanggup melakukan log ke file baru yang dirotate
# Kesimpulan
Mod_Security bekerja sebagai Web Application Firewall untuk menfilter request dari pemakai melalui predefined rule untuk mendeteksi eksploitasi WEB seperti upaya SqlInjection dan XSS maupun eksplotasi oleh pengembang dengan mengirim script yang beresiko ke sisi Client. Penerapan Rule terkait WAF membutuhkan proses penyesuaian sesuai dengan kondisi penerapan, sehingga kadang-kadang membutuhkan penyesuaian kembali seperti non-aktifkan beberapa rule berdasarkan ID, melakukan modifikasi pada regex expression, ataupun penyesuaian dari data dan system yang dikembangkan oleh developer.
