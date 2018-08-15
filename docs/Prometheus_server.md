Sơ đồ cài đặt tổng quát bao gồm một node Server và các node Exporter.

Trong đó, node Server cài đặt Prometheus server và các module khác như Alertmanager, Pushgateway, Grafana

Các node Exporter cài trên các máy cần monitor, tùy vào mục đích của từng máy để cài các Exporter hợp lý.

![Topo](https://raw.githubusercontent.com/locvx1234/ghichep-prometheus-v2/master/images/topo.png)

- [I. Cài đặt Prometheus server](#setup)
- [II. Giải thích cấu hình](#config)

<a name="setup"></a>
## I. Cài đặt Prometheus server

#### 0. Yêu cầu :
Ubuntu 16.04

Nginx 

```
sudo apt-get install nginx
sudo ufw allow 'Nginx HTTP'
```

#### 1. Tạo service user 

Tạo user `prometheus`:

```
sudo useradd --no-create-home --shell /bin/false prometheus
```

Cấu hình Prometheus trong `/etc` và data của nó sẽ lưu trong `/var/lib`     

```
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
```

Thiết lập ownership cho user `prometheus`:

```
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

#### 2. Download Prometheus

Tải các gói cài đặt từ trang chủ :     

```
cd ~
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.3.2/prometheus-2.3.2.linux-amd64.tar.gz
tar xvf prometheus-2.3.2.linux-amd64.tar.gz

sudo cp prometheus-2.3.2.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.3.2.linux-amd64/promtool /usr/local/bin/

sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool

sudo cp -r prometheus-2.3.2.linux-amd64/consoles /etc/prometheus
sudo cp -r prometheus-2.3.2.linux-amd64/console_libraries /etc/prometheus

sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries

rm -rf prometheus-2.3.2.linux-amd64.tar.gz prometheus-2.3.2.linux-amd64
```

#### 3. Cấu hình 

    sudo vi /etc/prometheus/prometheus.yml

##### Prometheus config file - `/etc/prometheus/prometheus.yml`

```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
```

    sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml

#### 4. Running Prometheus

Chạy thử :

```
sudo -u prometheus /usr/local/bin/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus/ \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries
```

Nếu lỗi thì xem lại file cấu hình

Tạm dừng bằng `CTRL+C` và sau đây sẽ tạo service trong `systemd`

    sudo vi /etc/systemd/system/prometheus.service

Copy nội dung vào file như sau:

##### Prometheus service file - `/etc/systemd/system/prometheus.service`

```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

Reload `systemd`:

```
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl status prometheus

sudo systemctl enable prometheus
```


#### 5. Securing 

```
sudo apt-get update
sudo apt-get install -y apache2-utils
```

Tạo tài khoản login (locvu) cho truy cập web:

    sudo htpasswd -c /etc/nginx/.htpasswd locvu

Config nginx:

```
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/prometheus
sudo vi /etc/nginx/sites-available/prometheus
```

##### File `/etc/nginx/sites-available/prometheus`
```
...
    location / {
        auth_basic "Prometheus server authentication";
        auth_basic_user_file /etc/nginx/.htpasswd;
        proxy_pass http://localhost:9090;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
...
```

```
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/prometheus /etc/nginx/sites-enabled/
```

Check config for error

    sudo nginx -t

    sudo systemctl reload nginx
    sudo systemctl status nginx


#### 6. Testing Prometheus

Truy cập: http://your_server_ip

Sau khi đăng nhập thành công, check status bằng cách vào Status > Target, ta thấy có 1 target là `prometheus`, nếu status của nó là `UP` thì thành công.

Còn nếu chưa `UP` thì vui rồi, check lại các service bằng lệnh:

```
sudo systemctl status prometheus
```

<a name="config"></a>
## II.Giải thích cấu hình

Prometheus cũng như bao ứng dụng khác, nó có thể sử dụng các đối số dòng lệnh và file cấu hình.

Để xem các lệnh khả dụng : 

```
prometheus -h
```

Prometheus có thể reload cấu hình của nó trong lúc chạy. Nếu cấu hình không đúng định dạng, sự thay đổi sẽ không được chấp nhận. 

File cấu hình 

Flag : `--config.file`

Định nghĩa một số kiểu giá trị: (để chú giải tiếng anh cho nó dễ hiểu :><)

```
<boolean>: a boolean that can take the values true or false
<duration>: a duration matching the regular expression [0-9]+(ms|[smhdwy])
<labelname>: a string matching the regular expression [a-zA-Z_][a-zA-Z0-9_]*
<labelvalue>: a string of unicode characters
<filename>: a valid path in the current working directory
<host>: a valid string consisting of a hostname or IP followed by an optional port number
<path>: a valid URL path
<scheme>: a string that can take the values http or https
<string>: a regular string
<secret>: a regular string that is a secret, such as a password
<tmpl_string>: a string which is template-expanded before usage
```

Ví dụ về file cấu hình mẫu : https://github.com/prometheus/prometheus/blob/release-2.3/config/testdata/conf.good.yml

Cấu hình global xác định các tham số có hiệu lực trong tất cả các ngữ cảnh cấu hình. Nó cũng như là mặc định đối với các phần khác. 

```
global:
  # Tần suất scrape targets
  [ scrape_interval: <duration> | default = 1m ]

  # Thời gian timeout của một request
  [ scrape_timeout: <duration> | default = 10s ]

  # Tần suất đánh giá các rule
  [ evaluation_interval: <duration> | default = 1m ]

  # Các lable để thêm vào time series hoặc alerts khi truyền thông với
  # các hệ thống mở rộng (federation, remote storage, Alertmanager).
  external_labels:
    [ <labelname>: <labelvalue> ... ]

# Các file rule xác định danh sách các glob. Rules và alerts đọc từ tất cả các file đó
rule_files:
  [ - <filepath_glob> ... ]

# Danh sách các scrape
scrape_configs:
  [ - <scrape_config> ... ]

# Các thiết lập liên quan tới Alertmanager.
alerting:
  alert_relabel_configs:
    [ - <relabel_config> ... ]
  alertmanagers:
    [ - <alertmanager_config> ... ]

# Các thiết lập liên quan tới tính năng remote write.
remote_write:
  [ - <remote_write> ... ]

# Các thiết lập liên quan tới tính năng remote read.
remote_read:
  [ - <remote_read> ... ]
```


#### <scrape_config>

Phần này xác định các target và các tham số cho việc mô tả cách scrape chúng. Thông thường, một cấu hình scrape là một job đơn. Cấu hình nâng cao thì có thể khác .

Các target có thể xác định qua tham số `static_configs` hoặc khám phá động, sử dụng service-discovery.

Thêm vào đó, `relabel_configs` cho phép thay đổi một vài target và gắn nhãn cho nó trước khi scrape.

```
# Tên job để scraped metrics.
job_name: <job_name>

# Tần suất scrape targets
[ scrape_interval: <duration> | default = <global_config.scrape_interval> ]

# Thời gian timeout khi scrape
[ scrape_timeout: <duration> | default = <global_config.scrape_timeout> ]

# The HTTP resource path on which to fetch metrics from targets.
[ metrics_path: <path> | default = /metrics ]

# Cấu hình này xem giải thích ở phần Pushgateway
[ honor_labels: <boolean> | default = false ]

# Giao thức sử dụng cho các request.
[ scheme: <scheme> | default = http ]

# Các tham số tùy chọn của HTTP URL.
params:
  [ <string>: [<string>, ...] ]

# Cài đặt `Authorization` header cho mọi scrape request sử dụng username/password.
# giá trị password và password_file sử dụng thay thế cho nhau.
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]

# Cài đặt `Authorization` header cho mọi scrape request sử dụng bearer token. 
# Nó thay thế `bearer_token_file`.
[ bearer_token: <secret> ]

# Cài đặt `Authorization` header cho mọi scrape request sử dụng bearer token file đọc từ file cấu hình. 
# Nó sử dụng thay cho `bearer_token`.
[ bearer_token_file: /path/to/bearer/token/file ]

# Cấu hình scrape request's TLS. tls_config:
[ <tls_config> ]

# Tùy chọn proxy URL.
[ proxy_url: <string> ]

# List of Azure service discovery configurations. azure_sd_configs:
[ - <azure_sd_config> ... ]

# List of Consul service discovery configurations. consul_sd_configs:
[ - <consul_sd_config> ... ]

# List of DNS service discovery configurations. dns_sd_configs:
[ - <dns_sd_config> ... ]

# List of EC2 service discovery configurations. ec2_sd_configs:
[ - <ec2_sd_config> ... ]

# List of OpenStack service discovery configurations. openstack_sd_configs:
[ - <openstack_sd_config> ... ]
...
# List of labeled statically configured targets for this job. static_configs:
[ - <static_config> ... ]

# List of target relabel configurations. relabel_configs:
[ - <relabel_config> ... ]

# List of metric relabel configurations. metric_relabel_configs:
[ - <relabel_config> ... ]

# Giới hạn số mẫu được chấp nhận khi scrape # 0 nghĩa là không giới hạn. [
sample_limit: <int> | default = 0 ]  
```

**Note**: <job_name> phải là duy nhất trong tất cả cấu hình scrape



#### <tls_config>

`tls_config` cho phép cấu hình các kết nối TLS

```
# CA certificate to validate API server certificate with.
[ ca_file: <filename> ]

# Certificate and key files for client cert authentication to the server.
[ cert_file: <filename> ]
[ key_file: <filename> ]

# ServerName extension to indicate the name of the server.
# http://tools.ietf.org/html/rfc4366#section-3.1
[ server_name: <string> ]

# Disable validation of the server certificate.
[ insecure_skip_verify: <boolean> ]
```


#### <static_config>
`static_config` cho phép xác định danh sách các target và một tập label cho chúng. 

```
# The targets specified by the static config.
targets:
  [ - '<host>' ]

# Labels assigned to all metrics scraped from the targets.
labels:
  [ <labelname>: <labelvalue> ... ]
```

#### <relabel_config>
Relabeling là công cụ mạnh mẽ cho việc tự động viết lại bộ label cho target trước khi nó được scrape. Nhiều bước relabeling có thể được cấu hình với mỗi cấu hình scrape. Chúng áp dụng cho bộ lablel của từng target theo thứ tự xuất hiện của chúng trong file cấu hình.

Phần tiếp [Exporter](Exporter.md)