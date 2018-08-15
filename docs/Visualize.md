
### 1. Grafana

Trước đây, Prometheus có giới thiệu PromDash sử dụng cho việc tạo Dashboad hiển thị kết quả các truy vấn. Nhưng hiện tại thì không còn được dùng nữa.

Trên trang chủ của Prometheus đã giới thiệu sử dụng Grafana là phương án thay thế

Sau đây là cách thêm Dashboard cho Prometheus, sử dụng template có sẵn: 


Thêm Data sources : Configuration > Data Sources > Add data source

![Add data source](https://raw.githubusercontent.com/locvx1234/ghichep-prometheus-v2/master/images/add_datasource.png)

Import template có sẵn : Chọn tab `Dashboards` và chọn `Import` cho những Dashboard có sẵn

![Template](https://raw.githubusercontent.com/locvx1234/ghichep-prometheus-v2/master/images/import_buildin.png)


Import tempate khác : Create > Import > Upload .json File

![Import](https://github.com/locvx1234/ghichep-prometheus-v2/blob/master/images/import.png)

Đây là một số file json mình sưu tập được 

[Link](../grafana-json)

Các bạn cũng có thể tự tạo các Dashboard theo ý muốn bằng cách sử dụng các truy vấn của Prometheus



