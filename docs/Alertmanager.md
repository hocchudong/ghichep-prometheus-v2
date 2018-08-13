Alerting và Prometheus tách thành 2 phần. Các Alerting rule được Prometheus gửi đến Alertmanager, Alertmanager quản lý việc cảnh báo, bao gồm  silencing, inhibition, aggregation và gửi cảnh báo đi qua các kênh như email, HipChat, PagerDuty.

3 bước để thiết lập cảnh báo:
- Cài đặt và cấu hình Alertmanager 
- Cấu hình Prometheus liên kết với Alertmanager 
- Tạo các rule alert trong Prometheus



Danh mục tìm hiểu:
- [0. Về Alertmanager](#alertmanager)
- [1. Cài đặt](#install)
- [2. Cấu hình](#config)
- [3. Cấu hình Prometheus trỏ đến Alertmanager](#config_server)
- [4. Tạo rule alert](#rule_alert)


<a name="alertmanager"></a>
### 0. Về Alertmanager 

Alertmanager xử lý cảnh báo được gửi bởi ứng dụng như là Prometheus server. Nó có các cơ chế Grouping, Inhibition, Silience

- Grouping :

Phân loại cảnh báo có những tính chất tương tự. Điều này thực sự hữu ích trong một hệ thống lớn với nhiều thông báo được gửi đông thời. 

Ví dụ: một hệ thống với nhiều server mất kết nối đến cơ sở dữ liệu, thay vì rất nhiều cảnh báo được gửi về Alertmanager thì Grouping giúp cho việc giảm số lượng cảnh báo trùng lặp, thay vào đó là một cảnh báo để chúng ta có thể biết được chuyện gì đang xảy ra với hệ thống của bạn.

- Inhibition :

Inhibition  là một khái niệm về việc chặn thông báo cho một số cảnh báo nhất định nếu các cảnh báo khác đã được kích hoạt. 

Ví dụ: Một cảnh báo đang kích hoạt, thông báo là cluster không thể truy cập (not reachable). Alertmanager có thể được cấu hình là tắt các cảnh báo khác liên quan đến cluster này nếu cảnh báo đó đang kích hoạt. Điều này lọc bớt những cảnh báo không liên quan đến vấn đề hiện tại. 

- Silience : 

Silience là tắt cảnh báo trong một thời gian nhất định. Nó được cấu hình dựa trên các match, nếu nó match với các điều kiện thì sẽ không có cảnh báo nào được gửi khi đó. 

- High avability:

Alertmanager hỗ trợ cấu hình để tạo một cluster với độ khả dụng cao. 

<a name="install"></a>
### 1. Cài đặt 

```
sudo useradd --no-create-home --shell /bin/false alertmanager
cd ~
curl -LO https://github.com/prometheus/alertmanager/releases/download/v0.15.1/alertmanager-0.15.1.linux-amd64.tar.gz
tar xvf alertmanager-0.15.1.linux-amd64.tar.gz

sudo mv alertmanager-0.15.1.linux-amd64/alertmanager /usr/local/bin
sudo mv alertmanager-0.15.1.linux-amd64/amtool /usr/local/bin

sudo chown alertmanager:alertmanager /usr/local/bin/alertmanager
sudo chown alertmanager:alertmanager /usr/local/bin/amtool

rm -rf alertmanager-0.15.1.linux-amd64 alertmanager-0.15.1.linux-amd64.tar.gz
```

<a name="config"></a>
### 2. Cấu hình

Alertmanager được cấu hình qua command-line flag và file cấu hình 

Để xem những flag khả dụng : `alertmanager -h`

#### 2.1 File cấu hình 

Để xác định file cấu hình được load : 
     
     alertmanager --config.file=simple.yml
 
Các kiểu dữ liệu :

- <duration>: a duration matching the regular expression [0-9]+(ms|[smhdwy])
- <labelname>: a string matching the regular expression [a-zA-Z_][a-zA-Z0-9_]*
- <labelvalue>: a string of unicode characters
- <filepath>: a valid path in the current working directory
- <boolean>: a boolean that can take the values true or false
- <string>: a regular string
- <secret>: a regular string that is a secret, such as a password
- <tmpl_string>: a string which is template-expanded before usage
- <tmpl_secret>: a string which is template-expanded before usage that is a secret

Ví dụ về một file cấu hình alert : https://github.com/prometheus/alertmanager/blob/master/doc/examples/simple.yml

Cấu hình global xác định tham số trong các bối cảnh khác nhau. Chúng là cấu hình mặc định nếu các phần cấu hình con không chỉ rõ.

```
global:
  # ResolveTimeout là thời gian tuyên bố đã được giải quyết nếu nó chưa được cập nhật
  [ resolve_timeout: <duration> | default = 5m ]

  # SMTP From header mặc định.
  [ smtp_from: <tmpl_string> ]
  # The default SMTP smarthost sử dụng cho việc gửi emails, bao gồm cả port number.
  # Port number thường là 25, hoặc 587 cho SMTP qua TLS (đôi khi gọi là STARTTLS).
  # Ví dụ: smtp.example.org:587
  [ smtp_smarthost: <string> ]
  # Hostname mặc định để định danh SMTP server.
  [ smtp_hello: <string> | default = "localhost" ]
  [ smtp_auth_username: <string> ]
  # SMTP Auth using LOGIN and PLAIN.
  [ smtp_auth_password: <secret> ]
  # SMTP Auth using PLAIN.
  [ smtp_auth_identity: <string> ]
  # SMTP Auth using CRAM-MD5. 
  [ smtp_auth_secret: <secret> ]
  # The default SMTP TLS requirement.
  [ smtp_require_tls: <bool> | default = true ]

  # API URL dùng để thông báo cho Slack
  [ slack_api_url: <string> ]
  [ victorops_api_key: <string> ]
  [ victorops_api_url: <string> | default = "https://alert.victorops.com/integrations/generic/20131114/alert/" ]
  [ pagerduty_url: <string> | default = "https://events.pagerduty.com/v2/enqueue" ]
  [ opsgenie_api_key: <string> ]
  [ opsgenie_api_url: <string> | default = "https://api.opsgenie.com/" ]
  [ hipchat_api_url: <string> | default = "https://api.hipchat.com/" ]
  [ hipchat_auth_token: <secret> ]
  [ wechat_api_url: <string> | default = "https://qyapi.weixin.qq.com/cgi-bin/" ]
  [ wechat_api_secret: <secret> ]
  [ wechat_api_corp_id: <string> ]

  # Cấu hình HTTP client mặc định
  [ http_config: <http_config> ]

# Các file template cho thông báo
# Có thể sử dụng wildcard matcher, vd. 'templates/*.tmpl'.
templates:
  [ - <filepath> ... ]

# The root node of the routing tree.
route: <route>

# Danh sách người nhận thông báo
receivers:
  - <receiver> ...

# Danh sách inhibition rules.
inhibit_rules:
  [ - <inhibit_rule> ... ]
```

#### 2.2 Cấu hình gửi qua email và slack 
Ví dụ về một file cấu hình :

File - /etc/alertmanager/alertmanager.yml
 
```
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'thuoclaoping@gmail.com'
  smtp_auth_username: 'thuoclaoping'
  smtp_auth_password: 'thu0c_la0'

  slack_api_url: 'https://hooks.slack.com/services/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'

route:
  group_by: [alertname, datacenter, app]
  receiver: 'team-1'

receivers:
  - name: 'team-1'
    email_configs:
    - to: 'locvx1234@gmail.com'
    slack_configs:
    - channel: '#alerts-checkmk'
      text: "<!channel> \nsummary: {{ .CommonAnnotations.summary }}\ndescription: {{ .CommonAnnotations.description }}"
```

#### 2.3 Chạy Alertmanager 
```
vi /etc/systemd/system/alertmanager.service
```

```
[Unit]
Description=Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
WorkingDirectory=/etc/alertmanager/
ExecStart=/usr/local/bin/alertmanager --config.file=/etc/alertmanager/alertmanager.yml --web.external-url http://your_server_ip:9093

[Install]
WantedBy=multi-user.target
```

<a name="config_server"></a>
### 3. Cấu hình Prometheus trỏ đến Alertmanager 

File - `/etc/prometheus/prometheus.yml`

```
...
rule_files:
  - alert.rules.yml

alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - localhost:9093
...
```

Sửa biến ExecStart trong file `/etc/systemd/system/prometheus.service`

```
ExecStart=/usr/local/bin/prometheus --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries \ 
    --web.external-url=http://your_server_ip
```

Restart Prometheus:

```
sudo systemctl daemon-reload
sudo systemctl restart prometheus
```

Khởi động Alertmanager :

```
sudo systemctl start alertmanager
sudo systemctl enable alertmanager
sudo ufw allow 9093/tcp
```

<a name="rule_alert"></a>
### 4. Tạo rule alert

```
sudo touch /etc/prometheus/alert.rules.yml
sudo chown prometheus:prometheus /etc/prometheus/alert.rules.yml
sudo vi /etc/prometheus/prometheus.yml
```

File - `/etc/prometheus/prometheus.yml`

```
global:
  scrape_interval: 15s

rule_files:
  - alert.rules.yml

scrape_configs:
...
```

Tạo một rule, rule này cảnh báo khi exporter không up 

```
sudo vi /etc/prometheus/alert.rules.yml
```

File - /etc/prometheus/alert.rules.yml

``` 
groups:
- name: Instances
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 5m
    labels:
      severity: page
    # Prometheus templates apply here in the annotation and label fields of the alert.
    annotations:
      description: '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes.'
      summary: 'Instance {{ $labels.instance }} down'
```

Check:
``` 
sudo promtool check rules /etc/prometheus/alert.rules.yml
```
 
Output
```
Checking /etc/prometheus/alert.rules.yml SUCCESS: 1 rules found
```
```
sudo systemctl restart prometheus
sudo systemctl status prometheus
```