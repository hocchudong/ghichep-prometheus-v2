## 1. Client library 

Bạn cần monitor một ứng dụng của bạn mà chưa có exporter hỗ trợ. Câu trả lời là Client library. Cơ chế tương tự như việc bạn đẩy metric và Pushgateway hoặc là viết một Exporter nhưng là tích hợp vào trong code của ứng dụng tùy thuộc đặc thù của dữ liệu.

Bạn sẽ phải định nghĩa các metric và phơi các giá trị của chúng qua HTTP endpoint trên ứng dụng của bạn. 

Các ngôn ngữ hỗ trợ :

- [Go](https://github.com/prometheus/client_golang)
- [Java or Scala](https://github.com/prometheus/client_java)
- [Python](https://github.com/prometheus/client_python)
- [Ruby](https://github.com/prometheus/client_ruby)

Bên cạnh đó có những [third-party client lib](https://prometheus.io/docs/instrumenting/clientlibs/) không chính thức

Khi Prometheus scrape instance của bạn, client library sẽ gửi tất cả các metric hiện tại tới server.

Mình quan tâm tới Python nên các phần bên dưới mình sẽ sử dụng Client_python để minh họa

### 2. Push metric 

Với những dữ liệu không thể scrape, nghĩa là phải đẩy về Pushgateway, bạn có thể sử dụng Bash shell như ví dụ ở phần [Pushgateway](Pushgateway.md) hoặc Client lib

Ví dụ :

```
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway

registry = CollectorRegistry()
g = Gauge('job_last_success_unixtime', 'Last time a batch job successfully finished', registry=registry)
g.set_to_current_time()
push_to_gateway('localhost:9091', job='batchA', registry=registry)
```
Giải thích ví dụ:

Trước tiên, sử dụng Registry riêng để chứa tạm các giá trị của metric. 

```
registry = CollectorRegistry()
```

Một metric `job_last_success_unixtime` được tạo với thuộc lớp Gauge

Gán giá trị cho metric là thời gian hiện tại

```
g.set_to_current_time()
```

hoặc cũng có thể gán một giá trị bất kỳ 
```
g.set(random.randint(1,101))
```

Cuối cùng là push tới Pushgateway, với label là job='batchA'

![Push client](https://raw.githubusercontent.com/locvx1234/ghichep-prometheus-v2/master/images/push_client.png)


### Viết Exporter 

// TODO




Phần tiếp [Visualize](Visualize.md)