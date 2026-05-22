# Chọn tên cho Task Queue

## Tên Task Queue

* Temporal Cluster duy trì một tập hợp các Task Queues.

* Workers sẽ poll các Task Queues để xem có work nào cần xử lý.

* Mỗi Task Queue được định danh bằng một name.

* Khi launch một Workflow Execution, Client sẽ truyền Task Queue name cho Temporal Cluster.

```go
options := client.StartWorkflowOptions{
    ID:        "my-workflow",
    TaskQueue: "my-task-queue-name",
}

run, err := c.ExecuteWorkflow(ctx, options, ProcessOrderWorkflow, input)
```

* Worker cũng phải được cấu hình để poll đúng Task Queue name đó.

```go
w := worker.New(c, "my-task-queue-name", worker.Options{})
```

## Vấn đề khi Task Queue name không khớp

* Task Queues được tạo động khi lần đầu được sử dụng.

* Vì vậy, nếu Client và Worker dùng hai Task Queue name khác nhau:

  * Temporal không báo lỗi trực tiếp.
  * Hai Task Queues khác nhau sẽ được tạo ra.
  * Worker sẽ không poll đúng Task Queue mà Workflow đang dùng.
  * Workflow Execution sẽ không thể tiếp tục chạy.

## Khuyến nghị: định nghĩa Task Queue name bằng constant

* Nên định nghĩa Task Queue name trong một constant.

* Cả Client và Worker cùng reference đến constant này để tránh typo hoặc mismatch.

```go
package app 

const TaskQueueName = "my-taskqueue-name"
```

### Client dùng constant khi start Workflow

```go
options := client.StartWorkflowOptions{
    ID:        "my-workflow",
    TaskQueue: app.TaskQueueName,
}

run, err := c.ExecuteWorkflow(ctx, options, ProcessOrderWorkflow, input)
```

### Worker dùng constant khi cấu hình

```go
w := worker.New(c, app.TaskQueueName, worker.Options{})
```

## Khi không thể dùng chung constant

* Không phải lúc nào cũng có thể dùng chung constant.

* Ví dụ:

  * Client chạy trên một system khác.
  * Client được viết bằng programming language khác.
  * Worker và Client không share cùng codebase.

## Chạy nhiều Worker Processes

* Trong production, nên chạy ít nhất 2 Worker Processes cho mỗi Task Queue.

* Lý do:

  * Tránh Worker trở thành single point of failure.
  * Nếu một Worker crash, Worker còn lại có thể recover các executions đang chạy.
  * Worker còn lại vẫn tiếp tục xử lý work mới.

* Chạy thêm nhiều Worker Processes giúp tăng:

  * Scalability.
  * Availability.

