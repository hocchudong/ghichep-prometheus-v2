Prometheus Pushgateway cung cấp các số liệu của các công việc tạm thời cho Prometheus server. Vì các công việc này tồn tại không đủ lâu để có thể scrape, thay vào đó thì các số liệu được đẩy đến Pushgateway, sau đó Pushgateway phơi các số liệu này cho Prometheus. 

Pushgateway rõ ràng không phải là một aggregator hoặc distributed counter mà đúng hơn là một metrics cache. Metric sẽ được push y như cách scrape một chương trình chạy vĩnh viễn. 

Với các metric machine-level, textfile collector của Node exporter thì thích hợp hơn, Pushgateway sử dụng cho các metric service-level. 

Danh mục tìm hiểu:

- [1. Cài đặt](#install)
- [2. Cấu hình](#config)
- [3. Về các nhãn : `job` và `instance`](#label)
- [4. Về timestamps](#timestamp)
- [5. URL](#url)


<a name="install"></a>
### 1. Cài đặt: 

- Source:

Build bằng source : https://github.com/prometheus/pushgateway . Cần setup Go và sử dụng Makefile 

- Binary:

Nếu bạn chạy với những thiết lập cơ bản, chỉ cần chạy file nhị phân. 
Download tại : https://github.com/prometheus/pushgateway/releases


Thay đổi địa chỉ listen : sử dụng cờ  (ví dụ :  "0.0.0.0:9091" hoặc ":9091").
Mặc định Pushgateway không duy trì metric, sử dụng cờ --persistence.file để xác định file sẽ duy trì metric (để chúng vẫn tồn tại khi restart Pushgateway).

- Docker 
```
docker pull prom/pushgateway
docker run -d -p 9091:9091 prom/pushgateway
```

<a name="config"></a>
### 2. Cấu hình: 

Pushgateway được cấu hình như target để scrape bởi Prometheus, sử dụng một trong các phương pháp thông thường. Tuy nhiên, bạn nên luôn đặt `honor_labels: true` trong file cấu hình. Ý nghĩa cấu hình này nằm ở bên dưới :))

Library: 

Client libraries cần có tính năng đẩy metric vào Pushgateway. Thông thường, một client được chỉ định các metric để scraping bởi Prometheus server một cách thụ động. Ở mã code phía client sẽ gọi đến hàm push để push metric đến Pushgateway qua API của nó. 

Command line 
Sử dụng Prometheus text protocol để push metric mà ko cần phải CLI riêng biệt nào hết. Đơn giản là sử dụng một command-line HTTP tool như `curl`

Chú ý : 

Với text protocol, kết thúc dòng sẽ là ký tự line-feed  ('LF' or '\n')
Kết thúc theo cách khác : 'CR' hay '\r', 'CRLF' hay còn viết là '\r\n' hoặc chỉ là kết thúc của gói tin thì kết quả sẽ lỗi


Ví dụ : Push một sample vào group được định danh bởi `{job="some_job"}`:

```
echo "second_metric 99" | curl --data-binary @- http://192.168.40.93:9091/metrics/job/some_job
```

Kết quả: 

![Job](https://raw.githubusercontent.com/locvx1234/ghichep-prometheus-v2/master/images/push_job.png)


Push nhiều thứ phức tạp hơn trong group định danh bởi `{job="some_job",instance="some_instance"}`:

```
cat <<EOF | curl --data-binary @- http://192.168.40.93:9091/metrics/job/some_job/instance/some_instance
# TYPE some_metric counter
some_metric{label="val1"} 42
# TYPE another_metric gauge
# HELP another_metric Just an example.
another_metric 2398.283
EOF
```

Chú thích : 

```
# TYPE : cung cấp loại thông tin 
# HELP : [option] giải thích ý nghĩa metric, rất được khuyến khích có để người khác còn hiểu chứ 
```

Kết quả:

![Job2](https://raw.githubusercontent.com/locvx1234/ghichep-prometheus-v2/master/images/push_job_detail.png)

Xóa các metric được group bởi job và instance:

```
curl -X DELETE http://pushgateway.example.org:9091/metrics/job/some_job/instance/some_instance
```

Xóa các metric được group bởi job

```
curl -X DELETE http://pushgateway.example.org:9091/metrics/job/some_job
```

<a name="label"></a>
### 3. Về các nhãn : `job` và `instance`

Prometheus server sẽ gắn các nhãn `job` và `instance` cho các metric được scrape. Giá trị của job lấy từ cấu hình scrape. Khi cấu hình Pushgateway như một scrape target, bạn có thể chọn một job name như là `pushgateway`. Giá trị của instance tự động gắn bằng host và port của target scrape. Do đó ở Prometheus server, tất cả các job đến từ Pushgateway là `pushgateway` và instance là `ip:port` của Pushgateway. Do đó, nó conflic với `job` và `instance` mà ta đang có. 

Điều này được giải quyết bằng cách, Prometheus sẽ chuyển đổi job metric thành và `exported_instance`. 

Viết dài dòng là vậy nhưng mà xem hình là hiểu :v

![exported instance](https://raw.githubusercontent.com/locvx1234/ghichep-prometheus-v2/master/images/exported_inststance.png)


Tuy nhiên, cách này không hay, khi mà chúng ta muốn giữ lại mấy cái label đó, tiện cho nhiều việc như filter chẳng hạn. 

Câu trả lời là: `honor_labels: true` được thiết lập trong phần cấu hình cho scrape từ pushgateway. 
 
Điều này lại gây ra một cái là có thể có những trường hợp mà không có nhãn instance. Điều này khá phổ biến vì các chỉ số thường ở service-level và không liên quan đến instance nào. Thực ra thì với `honor_labels: true`, Prometheus server sẽ gắn instance nếu không có instance nào được set. 

Do đó, nếu một metric được push đến Pushgateway mà không có nhãn instance (và không có nhãn trong grouping key), Pushgateway sẽ cho nó một nhãn instance rỗng (`{instance=""}`) tương đương với không có label nhưng ngăn được server đính kèm.

<a name="timestamp"></a>
### 4. Về timestamps:

Bạn push metric vào thời điểm t1, nhưng Prometheus gắn timestamp vào lúc nó scrape Pushgateway.

Bình thường, metric có thể scrape vào bất kỳ lúc nào. Metric mà không thể scrape về cơ bản là nó không còn tồn tại. Prometheus cho rằng, một metric mà không nhận được dữ liệu trong 5 phút thì nó sẽ coi như số liệu đó không còn nữa. Do vậy việc gắn timestamp khi push mà hơn 5 phút sau nó mới được scrape thì nó sẽ không được scrape, vì thế timestamp được gắn vào lúc scrape. 

Để tiện cho việc cảnh báo về các pusher không chạy gần đây, Pushgate có thêm metric là `push_time_seconds` với Unix timestamp của lần cuối cùng POST/PUT đối với mỗi group. Nó ghi đè giá trị metric trước đó. 

<a name="url"></a>
### 5. URL:

Port mặc định là `9091` với đường dẫn:

```
/metrics/job/<JOBNAME>{/<LABEL_NAME>/<LABEL_VALUE>}
```

`<JOBNAME>` là tên của nhãn job, tiếp theo là các cặp nhãn và giá trị của nó (có thể không có nhãn instance)
Bất kỳ nhãn nào trong phần body của request (như các nhãn thông thường, ví dụ `name{job="foo"} 42`) sẽ ghi đè các nhãn ở url. 

Lưu ý : `/` không sử dụng trong việc đặt name và value cho nhãn, ngay cả khi được thoát nghĩa `%2F`


##### PUT method: 

PUT được sử dụng để push một group metrics. Tất cả các metric với grouping key được xác định ở URL được thay thế bởi các metric được push với PUT.

##### POST method:

POST sử dụng giống như PUT, nhưng các metric cùng tên với các metric mới được push mới được thay thế (cùng grouping key)

##### DELETE method:

DELETE sử dụng để xóa metric từ Pushgateway.

Phần tiếp [Client lib](Client_lib.md)
