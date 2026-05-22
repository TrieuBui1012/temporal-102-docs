# Truy cập kết quả

## Accessing Results

* Temporal đảm bảo Durable Execution cho Workflow code.

* Vì vậy, có thể chạy các Workflows kéo dài nhiều tháng hoặc nhiều năm một cách đáng tin cậy.

* Các Activities bên trong Workflow Executions cũng có thể chạy trong thời gian rất dài.

## Execution trong Temporal là asynchronous

* Temporal hỗ trợ các execution dài hạn bằng cách làm cho chúng asynchronous so với code đã khởi tạo chúng.

* Ví dụ:

  * Client gọi `ExecuteWorkflow`.
  * Workflow gọi `ExecuteActivity`.

* Các lời gọi này không thực sự execute trực tiếp Workflow hoặc Activity.

* Thay vào đó, chúng gửi request đến Temporal Cluster để cluster thực hiện execution.

* Đây là các non-blocking calls.

* Flow điều khiển sẽ tiếp tục sang statement tiếp theo mà không cần chờ Workflow hoặc Activity Execution hoàn thành.

```go
// Use a client to request Workflow Execution
future, err := client.ExecuteWorkflow(ctx, options, MyWorkflow, input)

// Request Activity Execution from within a Workflow
future := workflow.ExecuteActivity(ctx, MyActivity, input)
```

* Các lời gọi trên chỉ submit execution requests đến cluster.

* Chúng không block để chờ execution hoàn thành.

# Chờ kết quả Execution

## Future là gì?

* Trong hầu hết trường hợp, code khởi tạo execution sẽ cần lấy kết quả mà execution trả về.

* Tuy nhiên, lời gọi khởi tạo phải return ngay lập tức.

* Kết quả thật sự chỉ có sau khi execution kết thúc.

* Vì vậy, nhiều Temporal APIs trả về một `Future`.

* `Future` cho phép truy cập kết quả của một asynchronous execution.

## Dùng `Get` để lấy kết quả

* Gọi `Get` trên `Future` để retrieve giá trị.

* `Get` sẽ block cho đến khi kết quả có sẵn ở cuối execution.

```go
// This chained call blocks until Activity Execution returns a result or error
var result string
err := workflow.ExecuteActivity(ctx, MyActivity, input).Get(ctx, &result)
```

* Cách viết này thường gặp trong Temporal application code.

* Statement trên kết hợp:

  * Request Activity Execution.
  * Retrieve result trong cùng một dòng.

* Vì phần sau gọi `Get` trên `Future` trả về từ phần đầu, nên statement này sẽ block cho đến khi execution hoàn tất.

# Trì hoãn việc lấy kết quả Execution

## Tách request execution và lấy result

* Nếu cần execute nhiều Activities độc lập với nhau, có thể giảm đáng kể tổng thời gian chạy của Workflow.

* Cách làm:

  * Gửi execution requests trước.
  * Lấy results sau.

* Nếu cluster và downstream systems có đủ capacity, các Activities có thể chạy song song.

## Ví dụ chạy nhiều Activities song song

```go
futureA := workflow.ExecuteActivity(ctx, MyActivityA, inputA)
futureB := workflow.ExecuteActivity(ctx, MyActivityB, inputB)
futureC := workflow.ExecuteActivity(ctx, MyActivityC, inputC)

// The following lines block until their respective executions have finished
var resultA string
errA := futureA.Get(ctx, &resultA)

var resultB string
errB := futureB.Get(ctx, &resultB)

var resultC string
errC := futureC.Get(ctx, &resultC)
```

## Lợi ích của cách này

* Ba Activities được request trước, sau đó mới chờ kết quả.

* Nếu ba Activities chạy trong thời gian tương đương nhau, Workflow có thể nhanh hơn khoảng 3 lần so với cách:

  * Chạy Activity A.
  * Chờ kết quả A.
  * Chạy Activity B.
  * Chờ kết quả B.
  * Chạy Activity C.
  * Chờ kết quả C.

* Đây là chiến lược tốt khi Workflow cần gọi các Activities không phụ thuộc lẫn nhau.
