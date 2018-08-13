## Prometheus là gì?

Prometheus là một hệ thống giám sát open-source mạnh mẽ dùng để thu thập các metric từ các dịch vụ và lưu trữ trong time-series database. Nó cung cấp một mô hinh dữ liệu đa chiều, ngôn ngữ linh hoạt và có thể hiển thị linh hoạt thông qua các công cụ như Grafana.

Nội dung chính : 

- [Tính năng](#feature)
- [Thành phần](#component)
- [Kiến trúc](#architecture)



<a name="feature"></a>
### Tính năng

Các tính năng chính:

- Data model đa chiều (time series data xác định bởi tên metric và cặp key/value)
- Ngôn ngữ truy vấn hiệu quả
- Không phụ thuộc lưu trữ phân tán, các server node là các autonomous
- Time series data được thu thập qua việc pull, sử dụng giao thức HTTP
- Các data không thể tổng hợp được push qua gateway trung gian
- Các target được phát hiện bằng cấu hình tĩnh hoặc nhờ service discovery
- Hỗ trợ dashboard, graph

<a name="component"></a>
### Thành phần 

Hệ sinh thái Prometheus gồm nhiều thành phần, nhiều trong số đó là những thành phần tùy chọn: 

- [Prometheus server](https://github.com/prometheus/prometheus) là thành phần chính, dùng để scrape và store time series data.
- [client libraries](https://prometheus.io/docs/instrumenting/clientlibs/) thư viện sử dụng để code trong ứng dụng của bạn
- [Pushgateway](https://github.com/prometheus/pushgateway) hỗ trợ các short-lived job
- [Exporter](https://prometheus.io/docs/instrumenting/exporters/) thu thập metric từ các serivce HAProxy, StatsD, Graphite, ... 
- [Alertmanager](https://github.com/prometheus/alertmanager) thành phần gửi cảnh báo
- và một vài công cụ hỗ trợ khác 

Chi tiết về từng thành phần, các hạ xem hồi sau sẽ rõ xD

<a name="architecture"></a>
### Kiến trúc

![Architecture](https://raw.githubusercontent.com/locvx1234/ghichep-prometheus-v2/master/images/architecture-cb2ada1ece6.png)

Prometheus scrape các metric từ các job/exporter hoặc từ pushgateway. Nó sẽ lưu trữ data và chạy các rule qua các data này để tổng hợp và ghi lại time series mới từ dữ liệu đã tồn tại hoặc là tạo cảnh báo. Grafana hoặc các API consumer có thể sử dụng để vẽ những dữ liệu thu thập được. 


#### Prometheus phù hợp khi nào?

Prometheus hoạt động tốt để ghi các time series data dạng số. Nó phù hợp với việc giám sát các máy chủ cũng như các kiến trúc highly dynamic service-oriented.

Đối với microservice, nó hỗ trợ thu thập và truy vấn dữ liệu đa chiều.

#### Prometheus không phù hợp khi nào?

Prometheus là đáng tin cậy. Bạn có thể xem thống kê nào có sẵn về hệ thống của bạn, ngay cả trong điều kiện lỗi. Nếu bạn cần độ chính xác 100%, ví dụ cho mục đích thanh toán, Prometheus không phải là một lựa chọn tốt vì nó không chi tiết và đầy đủ.


Phần tiếp [Prometheus server](Prometheus_server.md)