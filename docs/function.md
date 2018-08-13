Trong Prometheus query language, chúng ta có thể sử dụng các hàm được xây dựng sẵn để tạo ra các giá trị mong muốn 


Một số hàm có các đối số mặc định. Ví dụ: `year(v=vector(time()) instant-vector)`.

Nghĩa là, nếu nó có một đối số `v` nhận giá trị là một instance vector, nếu nó không được cung cấp thì sẽ nhận giá trị của biểu thức `vector(time())`

## abs()

`abs(v instant-vector)` trả về giá trị tuyệt đối của input

## absent()

`absent(v instant-vector)` trả về vector rỗng nếu 

## ceil()

`ceil(v instant-vector)` làm tròn các giá trị của các phần tử `v` lên số nguyên gần nhất

## changes()

`changes(v range-vector)` 