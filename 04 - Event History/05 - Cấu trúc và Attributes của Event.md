# Cấu trúc và Attributes của Event

## Event có nhiều loại khác nhau

* Một `Event History` có thể chứa nhiều loại `Events` khác nhau.

* Ví dụ:

```text
WorkflowExecutionStarted
ActivityTaskScheduled
```

* Temporal API hiện định nghĩa hơn 30 loại `Events` khác nhau.

## Mỗi Event có attributes riêng

* Mỗi loại `Event` sẽ có một hoặc nhiều `attributes`.

* Các `attributes` này dùng để lưu thông tin cụ thể cho loại `Event` đó.

* Ví dụ:

  * Error liên quan đến một `Activity` bị failed
  * Duration của một `Timer`

# Attributes chung của mọi Event

## Ba attributes luôn có trong mọi Event

* Mọi `Event` đều có ít nhất 3 attributes chung:

### Event ID

* `Event ID` định danh duy nhất một `Event` trong `History`.

* Đồng thời, `Event ID` cũng thể hiện vị trí của `Event` trong history.

### Event time

* `Event time` là timestamp cho biết thời điểm `Event` xảy ra.

### Event Type

* `Event Type` cho biết đây là loại `Event` nào.

* Ví dụ:

```text
WorkflowExecutionStarted
WorkflowExecutionCompleted
ActivityTaskScheduled
```

# Attributes thay đổi theo Event Type

## Không phải Event nào cũng có attributes giống nhau

* Ngoài 3 attributes chung, mỗi `Event Type` có thể có thêm các attributes riêng.

* Những attributes này phụ thuộc vào ý nghĩa của từng loại `Event`.

## Ví dụ với Workflow Events

### WorkflowExecutionStarted

* `WorkflowExecutionStarted` chứa thông tin như:

  * `Workflow Type`
  * Input data được truyền vào khi bắt đầu execution

### WorkflowExecutionCompleted

* `WorkflowExecutionCompleted` chứa:

  * Result của `Workflow Execution`

### WorkflowExecutionFailed

* Nếu Workflow failed, history sẽ kết thúc bằng:

```text
WorkflowExecutionFailed
```

* Event này chứa error được trả về từ execution.

## Ví dụ với Activity Events

### ActivityTaskScheduled

* `ActivityTaskScheduled` chứa thông tin như:

  * `Activity Type`
  * Input parameters truyền vào `Activity`

### ActivityTaskCompleted

* `ActivityTaskCompleted` chứa:

  * Result của `Activity Execution`

## Events liên quan đến Task scheduling

* Các `Events` liên quan đến Task scheduling thường chứa thông tin về:

  * Execution `Timeouts`
  * `Retry Policies`

* Những thông tin này giúp Temporal biết cách quản lý việc execute task, retry khi lỗi, và xử lý timeout.
