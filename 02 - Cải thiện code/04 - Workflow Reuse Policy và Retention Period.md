# Workflow ID Reuse Policy và Retention Period

## Workflow ID Reuse Policy là gì?

* Temporal đảm bảo trong cùng một `Namespace`, tại một thời điểm chỉ có một Workflow Execution đang chạy với một `Workflow ID`.

* Khi start Workflow, bạn có thể chỉ định thêm `Workflow ID Reuse Policy`.

* Policy này kiểm soát việc `Workflow ID` có được tái sử dụng bởi Workflow Execution khác trong cùng `Namespace` hay không.

## Các lựa chọn Workflow ID Reuse Policy

### `WORKFLOW_ID_REUSE_POLICY_ALLOW_DUPLICATE`

* Đây là policy mặc định.

* Cho phép `Workflow ID` được tái sử dụng sau khi Workflow Execution hiện tại kết thúc.

* Không quan trọng Workflow trước đó kết thúc thành công hay thất bại.

### `WORKFLOW_ID_REUSE_POLICY_ALLOW_DUPLICATE_FAILED_ONLY`

* Tương tự policy mặc định.

* Nhưng chỉ cho phép tái sử dụng `Workflow ID` nếu Workflow Execution trước đó **không hoàn thành thành công**.

* Phù hợp khi chỉ muốn chạy lại Workflow nếu lần trước thất bại.

### `WORKFLOW_ID_REUSE_POLICY_REJECT_DUPLICATE`

* Không cho phép tái sử dụng `Workflow ID`.

* Không quan trọng Workflow Execution trước đó kết thúc như thế nào.

### `WORKFLOW_ID_REUSE_POLICY_TERMINATE_IF_RUNNING`

* Tương tự policy mặc định, nhưng có thêm một điểm quan trọng.

* Nếu đang có Workflow Execution dùng `Workflow ID` đó, Temporal sẽ terminate Workflow đang chạy.

* Sau đó Workflow Execution mới sẽ là execution duy nhất đang chạy trong `Namespace` với `Workflow ID` đó.

## Chỉ định Workflow ID Reuse Policy

* Có thể chỉ định `Workflow ID Reuse Policy` bằng:

  * SDK.
  * Command-line option khi dùng `temporal` để start Workflow.

## Ví dụ với Go SDK

### Bước 1: import enum policy

* Cần thêm import sau để có thể reference các policy option theo tên.

```go
package main

import (
    // The following import is needed to reference the Workflow ID Reuse Policy value
    enums "go.temporal.io/api/enums/v1"
 
    // other imports removed for brevity
)
```

### Bước 2: thêm `WorkflowIDReusePolicy` vào options

```go
options := client.StartWorkflowOptions{
        ID:                    "example-workflow-id",
        TaskQueue:             shared.TaskQueueName,
        WorkflowIDReusePolicy: enums.WORKFLOW_ID_REUSE_POLICY_ALLOW_DUPLICATE_FAILED_ONLY,
}

we, err := c.ExecuteWorkflow(ctx, options, example.MyWorkflow)
// additional code would follow
```

* Ví dụ trên override policy mặc định.

* `Workflow ID` chỉ được tái sử dụng cho execution mới nếu execution trước đó kết thúc không thành công.

# Retention Period

## Retention Period là gì?

* Nếu dùng `Workflow ID Reuse Policy` nghiêm ngặt hơn, cần nhớ rằng Temporal chỉ enforce uniqueness dựa trên các Workflow Executions mà nó vẫn còn dữ liệu.

* Dữ liệu của Workflow Executions đã kết thúc sẽ được xóa sau một khoảng thời gian để:

  * Giữ hiệu năng tốt.
  * Tiết kiệm storage trong database của cluster.

* Khoảng thời gian từ lúc Workflow Execution kết thúc đến khi dữ liệu của nó đủ điều kiện bị xóa tự động gọi là `Retention Period`.

* `Retention Period` thường nằm trong khoảng từ 1 đến 30 ngày.

* Có thể cấu hình retention dài hơn nếu dùng version Temporal Cluster mới hơn, nhưng thường không khuyến khích vì:

  * Tăng nhu cầu storage.
  * Có thể làm giảm performance.

## Khi cần lưu dữ liệu lâu dài

* Một số hệ thống có yêu cầu compliance hoặc audit.

* Họ cần biết điều gì đã xảy ra trong Workflow Execution sau nhiều tháng hoặc nhiều năm.

* Tăng `Retention Period` có thể có vẻ hợp lý, nhưng Temporal cung cấp giải pháp khác phù hợp hơn.

## Archival trong self-hosted Temporal Cluster

* Với self-hosted Temporal cluster, có thể dùng tính năng `Archival`.

* Khi bật `Archival`:

  * Dữ liệu liên quan đến Workflow Execution sẽ được copy tự động sang long-term storage.
  * Ví dụ: local filesystem hoặc Amazon S3.
  * Sau đó dữ liệu trong cluster database sẽ bị xóa khi hết `Retention Period`.

## Export trong Temporal Cloud

* Temporal Cloud cung cấp tính năng tương tự tên là `Export`.

* Sau khi cấu hình `Export`, Temporal Cloud sẽ export history data từ `Namespace` sang S3 theo chu kỳ hằng giờ.

## Retention Period không ảnh hưởng đến Workflow đang chạy

* Countdown của `Retention Period` chỉ bắt đầu sau khi Workflow Execution kết thúc.

* `Retention Period` không ảnh hưởng đến các Workflow Executions đang chạy.

* Ví dụ:

  * `Retention Period` là 5 ngày.
  * Workflow Execution chạy trong 10 năm.
  * Workflow vẫn không bị ảnh hưởng.

* Dữ liệu của Workflow Execution đang chạy luôn có sẵn, bất kể execution chạy trong bao lâu.
