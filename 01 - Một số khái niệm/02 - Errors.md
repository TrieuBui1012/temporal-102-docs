# Error ảnh hưởng đến Workflow Execution như thế nào

* Trước đó, bạn đã thấy trường hợp execution chạy đúng như mong đợi:

  * Workflow Execution bắt đầu.
  * Workflow trả về result.
  * Workflow có thể request execute một hoặc nhiều Activities.
  * Mỗi Activity cũng trả về result.

* Nếu Workflow hoặc Activity trả về error thì sao?

* Trong cả hai trường hợp:

  * Chúng được xem là đã failed.

---

## Activity Errors

* Activities được dùng để đóng gói những phần dễ gặp failure trong Workflow.

* Ví dụ, bạn có thể dùng Activity để query database.

* Database query có thể fail vì nhiều lý do:

  * Database server từ chối credentials.
  * Database server đang offline khi mở connection.
  * SQL query có syntax error.

* Nếu Activity trả về error:

  * Temporal Cluster xem execution attempt đó là failed.

* Nếu execution có Retry Policy:

  * Retry Policy sẽ quyết định các lần retry tiếp theo được thực hiện như thế nào.

* Với Activity Execution, mặc định sẽ có Retry Policy.

* Khi dùng default Retry Policy:

  * Temporal tự động retry Activity.
  * Retry dùng exponential backoff.

* Sau failure đầu tiên:

  * Temporal chờ 1 giây trước retry đầu tiên.

* Nếu retry đó cũng fail:

  * Delay sẽ tăng gấp đôi ở các lần sau.
  * Delay tối đa mặc định là 100 giây.

* Temporal tiếp tục retry cho đến khi Activity:

  * Succeed.
  * Bị canceled.
  * Hoặc timed out.

* Điều này giúp xử lý tự động các lỗi tạm thời, ví dụ:

  * Temporary outages.
  * Network issue ngắn hạn.

* Với các lỗi không tự hết, ví dụ:

  * Incorrect credentials.
  * Invalid SQL query syntax.

* Bạn có thể sửa thủ công bằng cách thay đổi Activity code.

* Sau khi code được sửa, execution có thể succeed ở một attempt tiếp theo.

* Có thể thay đổi retry behavior bằng cách khai báo custom Retry Policy trong `workflow.ActivityOptions`.

* Mỗi lần gọi `workflow.ExecuteActivity` có thể dùng một Retry Policy khác nhau.

* Điều này giúp bạn linh hoạt implement business logic.

* Vì một lần gọi `workflow.ExecuteActivity` có thể khiến Activity code được execute nhiều hơn một lần, nên Activity code nên có tính **idempotent**.

* **Idempotent** nghĩa là:

  * Dù Activity được execute một lần hay retry nhiều lần.
  * State cuối cùng vẫn giống nhau.

---

## Workflow Errors

* Ngược lại với Activities, Workflows mặc định không có Retry Policy.

* Có thể gắn Retry Policy cho Workflow Execution.

* Tuy nhiên, việc này không phổ biến.

* Nếu Workflow trả về error:

  * Workflow sẽ được đánh dấu là failed.
  * Mặc định sẽ không được retry.

* Vì sao Activities được retry mặc định, còn Workflows thì không?

* Activities thường dùng để:

  * Thực hiện các operation dễ lỗi.
  * Tương tác với external systems.

* Các operation này dễ gặp intermittent failures, ví dụ:

  * Temporary network outages.

* Retry Activity thường có thể giúp execution succeed ở attempt tiếp theo.

* Workflows thì khác:

  * Workflow không trực tiếp exposed với các failure kiểu này.
  * Workflow thường chỉ fail nếu developer chủ động return error từ Workflow Definition.

* Trong Temporal, thường nên:

  * Fix nguyên nhân gốc gây error.
  * Thay vì return error và làm Workflow fail.

---

## Cách return errors từ application code

* Giống các application thông thường, bạn có thể return error khi gặp điều kiện bất lợi hoặc bất ngờ trong:

  * Workflow Definition.
  * Activity Definition.

* Ví dụ:

  * Một Activity gửi request đến Web server nên return error nếu request fail.

```go
resp, err := http.Get(url)
if err != nil {
    return "", errors.New("request failed")
}
```

* Có thể viết đơn giản hơn bằng cách return trực tiếp error nhận được:

```go
resp, err := http.Get(url)
if err != nil {
    // return the error from the failed HTTP request
    return "", err
}
```

* Developer không bắt buộc phải dùng Temporal-specific API để tạo hoặc return errors.

* Tuy nhiên, package `temporal` trong Go SDK có cung cấp một số function hỗ trợ.

* Ví dụ có thể dùng để:

  * Tạo error thuộc một type cụ thể.
  * Tạo error không được retry khi failure xảy ra.

* Errors được return từ application code sẽ được Temporal Worker tự động convert sang format trung lập với ngôn ngữ.

* Đây là một lý do giúp các phần khác nhau của application có thể được viết bằng các SDK khác nhau.

* Ví dụ:

  * TypeScript client execute Workflow viết bằng Go.
  * Workflow đó gọi một Activity viết bằng Python.
  * Và một Activity khác viết bằng Java.

* Mỗi ngôn ngữ có cách xử lý errors khác nhau.

* Cách Temporal xử lý error giúp:

  * Các ngôn ngữ interoperable với nhau.
  * Temporal Cluster serialize error details một cách nhất quán.
