# Các nguồn phổ biến gây non-determinism

## Ý chính

* `non-deterministic errors` không thể catch hoặc handle trực tiếp trong `Workflow code`.

* Cách tốt nhất là tránh viết `Workflow Definition` theo cách có thể tạo ra chuỗi `Commands` khác nhau giữa lần chạy thật và lần replay.

* Nếu logic cần yếu tố không deterministic hoặc cần tương tác bên ngoài, nên đưa phần đó vào `Activity`.

# Dùng random numbers

## Không nên dùng random trực tiếp trong Workflow

* Random number về bản chất là non-deterministic.

* Nếu random làm thay đổi control flow của Workflow, nó có thể làm thay đổi chuỗi `Commands`.

* Ví dụ nguy hiểm:

```go
if rand.Intn(100) >= 50 {
	workflow.Sleep(ctx, time.Hour * 4)
}
```

* Lần chạy đầu có thể tạo `StartTimer`.

* Lần replay có thể bỏ qua `workflow.Sleep`, khiến chuỗi `Commands` không khớp với `Event History`.

## Cách làm an toàn

* Tạo random number trong `Activity`.

* Return giá trị random từ `Activity` về `Workflow`.

* Vì result của `Activity` được lưu trong `Event History`, replay sẽ dùng lại cùng giá trị đó.

# Access hoặc mutate external systems/state

## Không truy cập trực tiếp hệ thống bên ngoài trong Workflow

* Ngoài việc gọi Temporal APIs, `Workflow code` không nên access hoặc mutate bất kỳ thứ gì bên ngoài chính nó.

* Những thứ cần tránh:

  * Global state
  * Database
  * File system
  * Input/output streams
  * Network services

## Vì sao nguy hiểm?

* External state có thể thay đổi giữa lần chạy ban đầu và lần replay.

* Nếu Workflow dựa vào state bên ngoài để quyết định gọi API nào, thứ tự `Commands` có thể bị thay đổi.

## Cách làm đúng

* Nếu business logic cần tương tác với external systems, hãy thực hiện trong `Activity`.

* `Workflow` chỉ gọi `Activity` và nhận result đã được ghi lại trong `Event History`.

# Dựa vào system time

## Không nên dùng system time trực tiếp

* Business logic trong Workflow không nên phụ thuộc trực tiếp vào system time.

* Không nên dùng các hàm như:

```go
time.Now
time.Sleep
```

* System time có thể khác nhau giữa lần chạy thật và lần replay.

## Dùng hàm an toàn của Temporal Go SDK

* Go SDK cung cấp các hàm an toàn cho Workflow.

### workflow.Now

* Dùng thay cho:

```go
time.Now
```

* Hàm an toàn:

```go
workflow.Now
```

* `workflow.Now` trả về system time tương ứng với thời điểm `Workflow Task` hiện tại được execute hoặc replay.

### workflow.Sleep

* Dùng thay cho:

```go
time.Sleep
```

* Hàm an toàn:

```go
workflow.Sleep
```

* `workflow.Sleep` schedule một durable `Timer`.

* Timer này trở thành một phần của `Workflow Execution Event History`.

# Làm việc trực tiếp với threads hoặc goroutines

## Không dùng goroutine trực tiếp trong Workflow

* OS chịu trách nhiệm schedule threads.

* Việc schedule này phụ thuộc vào resource availability và system load.

* Vì vậy, dùng trực tiếp threads hoặc goroutines trong `Workflow code` có thể dẫn đến execution không deterministic.

* Cần tránh:

```go
go func() {
	// ...
}()
```

## Dùng workflow.Go

* Nếu cần chạy goroutine trong Workflow, dùng API an toàn của Temporal:

```go
workflow.Go()
```

## Không dùng Go channels trực tiếp

* Không nên dùng trực tiếp Go channels và `select` trong Workflow.

* Cần tránh:

```go
select {
case v := <-ch:
	// ...
}
```

## Dùng workflow.Channel và workflow.Selector

* Temporal cung cấp các API an toàn hơn cho concurrency trong Workflow:

```go
workflow.Channel
workflow.Selector
```

# Iterate trên data structures có thứ tự không ổn định

## Cẩn thận khi iterate data structure

* Một số data structure không đảm bảo thứ tự iteration ổn định.

* Trong Go, ví dụ phổ biến là iterate qua `map` bằng `range`.

```go
for key, value := range someMap {
	// ...
}
```

## Vì sao nguy hiểm?

* Thứ tự iteration có thể thay đổi giữa lần chạy này và lần chạy khác.

* Nếu bên trong loop có gọi Temporal APIs, thứ tự `Commands` có thể thay đổi.

* Ví dụ:

```go
for _, item := range someMap {
	workflow.ExecuteActivity(ctx, ProcessItem, item).Get(ctx, nil)
}
```

* Nếu thứ tự map khác nhau khi replay, thứ tự schedule Activities cũng khác nhau.

## Cách làm an toàn

* Chuyển keys sang slice.

* Sort keys theo thứ tự ổn định.

* Iterate theo slice đã sort.

# Lưu hoặc đánh giá Run ID

## Workflow ID và Run ID

* Một số operation trong Temporal có thể tạo run mới cho cùng một `Workflow Execution`.

* Ví dụ:

```text
Continue-as-New
```

* Khi đó:

  * Các run có cùng `Workflow ID`.
  * Mỗi run có `Run ID` riêng.

## Không nên dùng Run ID trong logic Workflow

* Theo implementation hiện tại, `Run ID` là mutable.

* `Run ID` có thể thay đổi nếu Workflow được retry.

* Vì vậy, không nên:

  * Store `Run ID`
  * Dùng `Run ID` trong conditional statement

## Vì sao nguy hiểm?

* Nếu logic Workflow phụ thuộc vào `Run ID`, replay hoặc retry có thể đi theo nhánh khác.

* Điều này có thể làm thay đổi chuỗi `Commands` và gây `non-deterministic error`.