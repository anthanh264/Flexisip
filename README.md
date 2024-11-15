Flexisip is a complete and scalable SIP server suite that includes several modules: a proxy, a presence server, a conference server, a back-to-back user agent
# Deploy Flexisip for one domain
Domain: example.com
# Table of contents

- [Deploy Flexisip for one domain](#deploy-flexisip-for-one-domain)
  - [Domain Setup](#domain-setup)
  - [Server Setup](#server-setup)
    - [Cài đặt, cấu hình các phần mềm cần thiết](#cài-đặt-cấu-hình-các-phần-mềm-cần-thiết)
      - [Redis](#redis)
      - [MariaDB](#mariadb)
      - [Flexisip](#flexisip)
      - [TLS](#tls)
    - [Start Flexisip](#start-flexisip)
    - [Note](#note)
      - [Cấu hình ko sử dụng TLS](#cấu-hình-ko-sử-dụng-tls)
      - [Cấu hình sử dụng TLS](#cấu-hình-sử-dng-tls)
  - [References](#references)
## Domain Setup
- Cấu hình bản ghi DNS trên domain cần thực hiện 

| Host      | Type | Data          | TTL |
| --------- | ---- | ------------- | --- |
| test      | A    | <server_ip> | 120 |
| _sip._tcp | SRV  | 0 0 0 .       |   3600  |
|   _sip._udp        | SRV  | 0 0 0 .       |    3600 |
|   _sips._tcp        | SRV  | 0 0 0 .       |    3600 |

## Server Setup
- Server sử dụng Ubuntu 22.04 
- Update
```
apt update
apt upgrade -y
```
### Cài đặt, cấu hình các phần mềm cần thiết
#### Redis
- Cài đặt
```
sudo apt install redis-server -y

```
- Sửa cấu hình
```
sudo nano /etc/redis/redis.conf
```
Cuộn xuống và tìm tới dòng `supervised` và sửa thành `supervised systemd`
![image](https://i.imgur.com/0D6Jeus.png)

- Lưu và khởi động lại
```
sudo systemctl restart redis
```

#### MariaDB
- Cài đặt 
```
sudo apt install mariadb-server -y
```
- Tạo user và bảng trong csdl
```
mysql
CREATE DATABASE flexisip_accounts;
CREATE DATABASE flexisip_conference;
CREATE USER flexisip@localhost IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON flexisip_accounts.* TO flexisip@localhost;
GRANT ALL PRIVILEGES ON flexisip_conference.* TO flexisip@localhost;
```
- Thêm dữ liệu vào bảng
```
use flexisip_accounts;
CREATE TABLE accounts (
  id int(11) unsigned NOT NULL AUTO_INCREMENT,
  username varchar(64) NOT NULL,
  domain varchar(64) NOT NULL,
  password varchar(255) NOT NULL,
  algorithm varchar(10) NOT NULL DEFAULT 'MD5',
  PRIMARY KEY (id),
  UNIQUE KEY identity (username,domain)
);

INSERT INTO accounts VALUES
  (1,'user1','example.com','secret','CLRTXT'),
  (2,'user2','example.com','secret','CLRTXT'),
  (3,'user3','example.com','secret','CLRTXT');

exit;
```
- Sửa cấu hình
```
nano /etc/mysql/mariadb.conf.d/50-server.cnf
```
Tìm đến dòng 40 sửa thành `max_connections=500`
![image](https://i.imgur.com/oDv7YrY.png)
- Lưu và khởi động lại mysql
```
systemctl restart mariadb
```

#### Flexisip
- Tạo Belledonne Communications APT repository
```
echo "# For Ubuntu 22.04 LTS
deb [arch=amd64] http://linphone.org/snapshots/ubuntu jammy stable # hotfix beta alpha" | sudo tee /etc/apt/sources.list.d/belledonne.list

```
- Cài  Belledonne Communications PGP key
```
wget https://www.linphone.org/snapshots/ubuntu/pubkey.gpg -O - | sudo apt-key add -
```
- Cài packages dành cho SQL back-end
```
apt install libmariadb-dev -y
```
- Cập nhập APT-cache và cài Flexisip
```
apt update
apt install bc-flexisip -y
```
- Sửa cấu hình 
Backup file flexisip.conf 
```
cd /etc/flexisip/
mv flexisip.conf flexisip.conf.backup
```
Tạo file mới và paste nội dung sau 
```
nano flexisip.conf 
```
```
## PROXY SETTINGS ##
[global]
log-level=debug
syslog-level=debug
# Use TLS only for public communications and
# TCP for connections with services running
# on localhost (conference server, presence server, ...)
#transports=sips:0.0.0.0:5061 sip:127.0.0.1:5060;transport=tcp
transports=sips:<server_ip>:5061;maddr=10.0.0.66
#transports=sip:<server_ip>:5060;maddr=10.0.0.66
tls-certificates-file=/etc/letsencrypt/live/example.com/fullchain.pem
tls-certificates-private-key=/etc/letsencrypt/live/example.com/privkey.pem

# Enable digest authentication
[module::Authentication]
enabled=true
auth-domains=example.com
available-algorithms=SHA-256
trusted-hosts=127.0.0.1
db-implementation=soci
soci-backend=mysql
soci-connection-string=db='flexisip_accounts' user='flexisip' password='password' host='localhost'
soci-password-request=select password, algorithm from accounts where username= :id and domain= :domain;

# Enable Registrar feature by using the Redis database
[module::Registrar]
enabled=true
reg-domains=example.com
db-implementation=redis
redis-server-domain=localhost
redis-server-port=6379

# Allow user agents to be registered for
# seven days in order they can receive
# push notifications seven days after
# disconnection.
max-expires=604800



[module::Router]
# Enable fork-late because it is required
# to send push notifications.
fork-late=true

# Chat messages will be kept by the proxy
# for seven days at the most if it cannot
# be delivered to the recipient immediately.
message-delivery-timeout=604800

# Enable push notifications for iOS and Android clients.
[module::PushNotification]
enabled=false
apple=true
firebase=true
firebase-projects-api-keys=<your_project_api_key>


## PRESENCE SERVER SETTINGS ##
[presence-server]
enabled=true
transports=sip:127.0.0.1:5065;transport=tcp

[module::Presence]
enabled=true
presence-server=sip:127.0.0.1:5065;transport=tcp


## CONFERENCE SERVER SETTINGS ##
[conference-server]
enabled=true
transport=sip:127.0.0.1:6064;transport=tcp
conference-factory-uris=sip:conference-factory@example.com
outbound-proxy=sip:127.0.0.1:5060;transport=tcp
local-domains=example.com
database-backend=mysql
database-connection-string=db='flexisip_conference' user='flexisip' password='password' host='localhost'

```
#### TLS
- Cài đặt Apache và Certbot
```
sudo apt install certbot python3-certbot-apache -y
```
- Tạo thư mục web cho subdomain 
```
mkdir -p /var/www/example.com/public_html
```
- Gán quyền 
```
chown -R www-data:www-data /var/www/example.com/public_html
```
- Tạo file config host
```
nano /etc/apache2/sites-available/example.com.conf
```
Và thêm nội dung sau 
```
<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com
    ServerAdmin admin@domain.com
    DocumentRoot /var/www/example.com/public_html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
- Enable Host mới tạo 
```
a2ensite example.com.conf
```
- Restart apache2
```
service apache2 restart
```
- Sinh chứng chỉ SSL 
```
sudo certbot --apache
```
- Tập lệnh này sẽ yêu cầu bạn trả lời một loạt các câu hỏi để cấu hình chứng chỉ SSL của bạn. Đầu tiên, nó sẽ yêu cầu bạn nhập một địa chỉ email hợp lệ. Địa chỉ email này sẽ được sử dụng cho thông báo gia hạn và thông báo bảo mật:
```
Output
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): you@your_domain
```
- Sau khi cung cấp một địa chỉ email hợp lệ, nhấn ENTER để tiếp tục bước tiếp theo. Bạn sau đó sẽ được yêu cầu xác nhận liệu bạn đồng ý với các điều khoản dịch vụ của Let's Encrypt. Bạn có thể xác nhận bằng cách nhấn Y và sau đó nhấn ENTER:
```
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
agree in order to register with the ACME server at
https://acme-v02.api.letsencrypt.org/directory
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y
```
- Tiếp theo, bạn sẽ được hỏi liệu bạn có muốn chia sẻ địa chỉ email của mình với Electronic Frontier Foundation để nhận tin tức và thông tin khác hay không. Nếu bạn không muốn đăng ký nội dung của họ, hãy nhập N. Nếu không, nhập Y sau đó nhấn ENTER để tiếp tục bước tiếp theo:
```
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing to share your email address with the Electronic Frontier
Foundation, a founding partner of the Let's Encrypt project and the non-profit
organization that develops Certbot? We'd like to send you email about our work
encrypting the web, EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: N
```
- Bước tiếp theo sẽ yêu cầu bạn thông báo cho Certbot biết bạn muốn kích hoạt HTTPS cho những tên miền nào. Các tên miền được liệt kê sẽ tự động được lấy từ cấu hình máy chủ ảo Apache của bạn. Trong bài lab này là example.com
```
Which names would you like to activate HTTPS for?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: your_domain
2: www.your_domain
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate numbers separated by commas and/or spaces, or leave input
blank to select all options shown (Enter 'c' to cancel): 
```
- Sau bước này, cấu hình của Certbot đã hoàn tất, và bạn sẽ được hiển thị các ghi chú cuối cùng về chứng chỉ mới của bạn và nơi có thể tìm thấy các tệp đã tạo:
```
Output
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/your_domain/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/your_domain/privkey.pem
This certificate expires on 2022-07-10.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

Deploying certificate
Successfully deployed certificate for your_domain to /etc/apache2/sites-available/your_domain-le-ssl.conf
Successfully deployed certificate for www.your_domain.com to /etc/apache2/sites-available/your_domain-le-ssl.conf
Congratulations! You have successfully enabled HTTPS on https:/your_domain and https://www.your_domain.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```
- Sau bước này ta thu được chứng chỉ SSL/TLS ở thư mục `/etc/letsencrypt/live/example.com`

![image](https://i.imgur.com/anzxLUQ.png)

### Start Flexisip
- Start the service
```
systemctl start flexisip-{proxy,presence,conference}
```

### Note
#### Cấu hình ko sử dụng TLS
- Sửa file flexisip.conf
```
nano flexisip.conf
```
Comment nhưng dòng sau
`transports=sips:<server_ip>:5061;maddr=10.0.0.66`

```
tls-certificates-file=/etc/letsencrypt/live/example.com/fullchain.pem
tls-certificates-private-key=/etc/letsencrypt/live/example.com/privkey.pem
```
Bỏ Comment  `transports=sip:<server_ip>:5060;maddr=10.0.0.66`

- Restart lại service
```
systemctl restart flexisip-{proxy,presence,conference}
```
#### Cấu hình sử dụng TLS 
- Làm ngược lại bước trên. 


## References
- [Installation Flexisip](https://wiki.linphone.org/xwiki/wiki/public/view/Flexisip/1.%20Installation/#HPost-installationinstructions)
- [Deploy Flexisip for one domain](https://wiki.linphone.org/xwiki/wiki/public/view/Flexisip/HOWTOs/Deploy%20Flexisip%20for%20one%20domain/)
- [Cài đặt Let's Encrypt SSL trên Ubuntu 22.04 với Apache và sử dụng Certbot](https://kdata.vn/cam-nang/cai-dat-lets-encrypt-ssl-tren-ubuntu-2204-voi-apache-va-su-dung-certbot)
- [How to add multiple subdomains to an Apache server?](https://www.studytonight.com/apache-guide/how-to-add-multiple-subdomains-to-an-apache-server)
- [How to Install Redis on Ubuntu 22.04 in 5 Steps [With Examples]](https://www.cherryservers.com/blog/install-redis-ubuntu)
- [[Linphone-developers] Flexisip/no voice or video.](https://linphone-developers.nongnu.narkive.com/t4Bw3Byi/flexisip-no-voice-or-video)
- [Imgur gallery](https://imgur.com/a/2gQ6ybx)
