## I. Về các exporter

Theo mặc định, Prometheus chỉ thu thập những số liệu về chính nó (ví dụ số yêu cầu nhận, lượng bộ nhớ tiêu thụ,...). Để có thể mở rộng thu thập từ các nguồn khác, sử dụng Exporter và các công cụ khác để tạo các metric. 

Exporters - được phát triển bởi Prometheus và cộng đồng - cung cấp mọi thứ thông tin về cơ sở hạ tầng, cơ sở dữ liệu, web server, hệ thống tin nhắn, API, ...

#### Một vài Exporter tiêu biểu:

- **node_exporter** : tạo ra các số liệu về hạ tầng, bao gồm CPU, memory, disk usage cũng như số liêu I/O, network.
- **blackbox_exporter** : tạo ra các số liệu từ các đầu dò như HTTP, HTTPs để xác định tính khả dụng của các endpoint, thời gian phản hồi,...
- **mysqld_exporter** : tập hợp các số liệu liên quan tới mysql server  
- **rabbitmq_exporter** : output của exporter này liên quan tới RabbitMQ, bao gồm số lượng message được publish, số message sẵn sàng để gửi, kích cỡ các gói tin trong hàng đợi.
- nginx-vts-exporter : cung cấp các số liệu về nginx server sử dụng module VTS bao gồm số lượng kết nối mở, số lượng phản hồi được gửi và tổng kích thước của các gói tin gửi và nhận

Các exporter xem thêm tại : https://prometheus.io/docs/instrumenting/exporters/

#### Metric

Định dạng của một metric có dạng : 

```
<metric name>{<label name>=<label value>, ...}
```

Mỗi exporter sẽ chạy phơi data mà nó thu thập được để Prometheus server có thể pull về qua giao thức http.

Thông thường, chúng ta có thể xem trực tiếp các metric này ở địa chỉ `ip-exporter:port/metrics`

Ví dụ một đoạn metric tôi lấy ra từ `localhost:9100/metrics` 


```
...
# HELP node_network_receive_bytes_total Network device statistic receive_bytes.
# TYPE node_network_receive_bytes_total counter
node_network_receive_bytes_total{device="ens3"} 7.086626e+06
node_network_receive_bytes_total{device="lo"} 654508
# HELP node_network_receive_compressed_total Network device statistic receive_compressed.
# TYPE node_network_receive_compressed_total counter
node_network_receive_compressed_total{device="ens3"} 0
node_network_receive_compressed_total{device="lo"} 0
# HELP node_network_receive_drop_total Network device statistic receive_drop.
# TYPE node_network_receive_drop_total counter
node_network_receive_drop_total{device="ens3"} 77
node_network_receive_drop_total{device="lo"} 0
...
```

Các dòng có dấu `#` chỉ để chú thích

- `# HELP`: giải thích ý nghĩa của metric
- `# TYPE`: loại giá trị

Có 4 TYPE:

- **counter**: là một bộ đếm tích lũy, được đặt về 0 khi restart. Ví dụ, có thể dùng `counter` để đếm số request được phục vụ, số lỗi, số task hoàn thành,... Không sử dụng cho gía trị có thể giảm như số tiến trình đang chạy. Trong trường hợp này, ta có thể sử dụng `gauge`

- **gauge**: đại diện cho số liệu duy nhất, nó có thể lên hoặc xuống. Nó thường được sử dụng cho các giá trị đo
- **histogram**: lấy mẫu quan sát (thường là những thứ như là thời lượng yêu cầu, kích thước phản hồi). Nó cũng cung cấp tổng của các giá trị đó.
- **summary**: tương tự histogram, nó cung cấp tổng số các quan sát và tổng các giá trị đó, nó tính toán số lượng có thể  cấu hình qua một cửa sổ trượt


Một số metric phổ biến mình đang cập nhập tại đây: 

https://docs.google.com/spreadsheets/d/1tsHNQfAMfTlI-RELY-HWwbnE2QUH6oFEpFXGiY6LZGA/edit?usp=sharing

## II. Cài đặt

Chi tiết cách cài đặt của một số Exporter sẽ được cập nhật tại đây: 

- [1. Node Exporter](#node)
- [2. Blackbox Exporter](#blackbox)
- [3. Mysqld exporter](#mysql)


<a name="node"></a>
### 1. Node Exporter

#### Trên máy cần monitor: 

Tạo user `node_exporter`:

```
sudo useradd --no-create-home --shell /bin/false node_exporter
```

Tải bản mới nhất từ [trang chủ](https://prometheus.io/download/) : 

```
cd ~
curl -LO https://github.com/prometheus/node_exporter/releases/download/v0.16.0/node_exporter-0.16.0.linux-amd64.tar.gz

tar xvf node_exporter-0.16.0.linux-amd64.tar.gz
sudo cp node_exporter-0.16.0.linux-amd64/node_exporter /usr/local/bin
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter

rm -rf node_exporter-0.16.0.linux-amd64.tar.gz node_exporter-0.16.0.linux-amd64
```


Running node exporter

```
sudo vi /etc/systemd/system/node_exporter.service
```

Node Exporter service file - `/etc/systemd/system/node_exporter.service`
```
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reload
sudo systemctl start node_exporter

sudo systemctl enable node_exporter
```

#### Trên máy Prometheus server:

Cấu hình Prometheus server để scrape Node Exporter 

```
sudo vi /etc/prometheus/prometheus.yml
```

Prometheus config file - /etc/prometheus/prometheus.yml
```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node_exporter'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100']       
```

```
sudo systemctl restart prometheus
sudo systemctl status prometheus
```

Test 


Check xem Node exporter có chạy đúng ko :

Truy cập địa chỉ Promethes, check status của target `node_exporter` đảm bảo rằng nó `UP`

Click vào tab Graph trong menu bar

Trong ô Expression, gõ `node_memory_MemAvailable_bytes` và chọn `Execute`

![MemAvailable](https://raw.githubusercontent.com/locvx1234/ghichep-prometheus-v2/master/images/node_memory_Mem.png)


Bình thường là kết quả nó sẽ trả về dạng `byte`, bạn muốn kết quả trả về dạng Megabyte,
đơn giản với một phép toán như sau :  `node_memory_MemAvailable_bytes/1024/1024`


Để kiểm chứng kết quả, check memory máy cài Node exporter:

``` 
free -h
```

Bonus : Ngôn ngữ truy vấn Prometheus cung cấp một hàm cho việc tổng hợp kết quả.

vd : `avg_over_time(node_memory_MemAvailable_bytes[5m])/1024/1024` : Available Memory trung bình trong 5 phút 

Xem thêm các hàm tại : https://prometheus.io/docs/prometheus/latest/querying/functions/


<a name="blackbox"></a>
### 2. Blackbox exporter


#### Trên máy cần monitor: 

Tạo user `blackbox_exporter`:

```
sudo useradd --no-create-home --shell /bin/false blackbox_exporter
```

Tải bản mới nhất từ [trang chủ](https://prometheus.io/download/): 

```
cd ~
curl -LO https://github.com/prometheus/blackbox_exporter/releases/download/v0.12.0/blackbox_exporter-0.12.0.linux-amd64.tar.gz

tar xvf blackbox_exporter-0.12.0.linux-amd64.tar.gz
sudo mv ./blackbox_exporter-0.12.0.linux-amd64/blackbox_exporter /usr/local/bin
sudo chown blackbox_exporter:blackbox_exporter /usr/local/bin/blackbox_exporter

rm -rf ~/blackbox_exporter-0.12.0.linux-amd64.tar.gz ~/blackbox_exporter-0.12.0.linux-amd64
```

Config blackbox exporter
```
sudo mkdir /etc/blackbox_exporter
sudo chown blackbox_exporter:blackbox_exporter /etc/blackbox_exporter
sudo vi /etc/blackbox_exporter/blackbox.yml
```

File `/etc/blackbox_exporter/blackbox.yml`

```
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:      
      valid_status_codes: []
      method: GET
```

```
sudo chown blackbox_exporter:blackbox_exporter /etc/blackbox_exporter/blackbox.yml
```

Running blackbox exporter

```
sudo vi /etc/systemd/system/blackbox_exporter.service
```

Blackbox Exporter service file - `/etc/systemd/system/blackbox_exporter.service`
```
[Unit]
Description=Blackbox Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=blackbox_exporter
Group=blackbox_exporter
Type=simple
ExecStart=/usr/local/bin/blackbox_exporter --config.file /etc/blackbox_exporter/blackbox.yml

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reload
sudo systemctl start blackbox_exporter

sudo systemctl enable blackbox_exporter
```

#### Trên máy Prometheus server:  

Cấu hình Prometheus server để scrape Node Exporter 

```
sudo vi /etc/prometheus/prometheus.yml
```

Prometheus config file - /etc/prometheus/prometheus.yml
```
...
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - http://localhost:8080
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115
```

```
sudo systemctl restart prometheus
sudo systemctl status prometheus
```



<a name="mysql"></a>
### 3. Mysqld exporter


#### Trên máy cần monitor: 

Tạo user `mysql_exporter`:

```
sudo adduser --no-create-home --disabled-login --shell /bin/false --gecos "MySQL exporter user" mysql_exporter
```

Tải bản mới nhất từ [trang chủ](https://prometheus.io/download/): 

```
cd ~
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.11.0/mysqld_exporter-0.11.0.linux-amd64.tar.gz

tar xvf mysqld_exporter-0.11.0.linux-amd64.tar.gz

sudo mv ./mysqld_exporter-0.11.0.linux-amd64/mysqld_exporter /usr/local/bin/
sudo chown mysql_exporter:mysql_exporter /usr/local/bin/mysqld_exporter

rm -rf ~/mysqld_exporter-0.11.0.linux-amd64.tar.gz ~/mysqld_exporter-0.11.0.linux-amd64
```

Config mysqld exporter
```
sudo mkdir /etc/mysql_exporter
sudo touch /etc/mysql_exporter/.my.cnf

sudo chown -R mysql_exporter:mysql_exporter /etc/mysql_exporter
sudo chmod 600 /etc/mysql_exporter/.my.cnf
sudo vi /etc/mysql_exporter/.my.cnf
```

File `/etc/mysql_exporter/.my.cnf`

```
[client]
user=REPLACE_WITH_YOUR_USER
password=REPLACE_WITH_YOUR_PASSWORD
```

Running mysqld exporter

```
sudo vi /etc/systemd/system/mysql_exporter.service
```

Mysqld Exporter service file - `/etc/systemd/system/mysql_exporter.service`
```
[Unit]
Description=Prometheus MySQL Exporter
After=network.target

[Service]
User=mysql_exporter
Group=mysql_exporter
Type=simple
ExecStart=/usr/local/bin/mysqld_exporter \
    --config.my-cnf="/etc/mysql_exporter/.my.cnf"
Restart=always

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reload
sudo systemctl enable mysql_exporter
sudo systemctl start mysql_exporter
```

#### Trên máy Prometheus server:  

Cấu hình Prometheus server để scrape Node Exporter 

```
sudo vi /etc/prometheus/prometheus.yml
```

Prometheus config file - /etc/prometheus/prometheus.yml
```
...
  - job_name: 'mysql_exporter'
    scrape_interval: 5s
    static_configs:
      - targets:
        - localhost:9104
```

```
sudo systemctl restart prometheus
sudo systemctl status prometheus
```

Phần tiếp [Alertmanager](Alertmanager.md)
