# Chọn Workflow ID

## Workflow ID là gì?

* Khi start một Workflow, bạn cần chỉ định một `Workflow ID`.

* `Workflow ID` được gắn với Workflow Execution đó.

* `Workflow ID` nên có ý nghĩa với business logic.

* Ví dụ:

  * Workflow xử lý đơn hàng có thể dùng order number trong `Workflow ID`.

    * `process-order-number-90743812`
  * Workflow quản lý khoản vay có thể dùng account number.

    * `manage-loan-account-28430614`

```go
// Example: An order processing Workflow might include order number in the Workflow ID
options := client.StartWorkflowOptions{
    ID:        "process-order-number-"   input.OrderNumber,
    TaskQueue: app.TaskQueueName,
}

run, err := c.ExecuteWorkflow(ctx, options, ProcessOrderWorkflow, input)
```

## Tính duy nhất của Workflow ID

* Temporal đảm bảo rằng trong cùng một namespace, tại một thời điểm chỉ có duy nhất một Workflow Execution đang chạy với cùng một `Workflow ID`.

* `namespace` là một logical unit of isolation.

* Namespace thường được dùng để tách biệt các Workflows:

  * Chạy trên cùng một cluster.
  * Nhưng thuộc về các team hoặc domain khác nhau.

## Workflow ID phải duy nhất trong toàn namespace

* Ràng buộc về `Workflow ID` áp dụng cho mọi loại Workflow Execution trong cùng namespace.

* Không chỉ áp dụng cho các Workflow cùng type.

* Ví dụ:

  * Một order processing Workflow đang chạy với một `Workflow ID`.
  * Một loan management Workflow khác không thể dùng cùng `Workflow ID` đó nếu execution kia vẫn đang chạy.

* Đây là điểm quan trọng cần cân nhắc khi chọn giá trị cho `Workflow ID`.

## Temporal xử lý xung đột Workflow ID như thế nào?

* Trong Go SDK, nếu cố start một Workflow Execution mới với `Workflow ID` đã có Workflow khác đang chạy:

  * Workflow mới sẽ không được start.
  * Request gần như được bỏ qua.
  * SDK sẽ trả về thông tin của Workflow Execution đang chạy sẵn.

* Có thể cấu hình `StartWorkflowOptions` để thay vì trả về Workflow đang chạy sẵn, SDK sẽ trả về error khi xảy ra conflict.
