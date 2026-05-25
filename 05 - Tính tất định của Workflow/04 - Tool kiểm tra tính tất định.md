# Xác định code non-deterministic bằng Static Analysis

## workflowcheck là gì?

* Với Go Workflows, có thể dùng command-line tool:

```text
workflowcheck
```

* Tool này giúp phát hiện nhiều loại code non-deterministic trong `Workflow`.

* Cách hoạt động:

  * Tìm các `Workflow Definitions` đã được register với `Worker`.
  * Static analyze phần code tương ứng.
  * Báo ra các đoạn code có nguy cơ vi phạm determinism.

# Giới hạn của workflowcheck

## Không phát hiện được mọi lỗi

* `workflowcheck` rất hữu ích để bắt lỗi nhanh, nhưng không thể phát hiện tất cả vi phạm non-deterministic.

* Một số trường hợp tool có thể bỏ sót:

  * Mutate global variables
  * Operations thực hiện qua reflection

## Có thể có false positives

* Một số version cũ của tool có thể báo lỗi nhầm với các function thực ra an toàn trong `Workflow code`.

* Ví dụ:

```text
fmt.Sprintf
fmt.Append
```

# Cài đặt workflowcheck

## Dùng go install

* Cài bản mới nhất bằng lệnh:

```bash
go install go.temporal.io/sdk/contrib/tools/workflowcheck@latest
```

* Sau khi chạy lệnh trên, `workflowcheck` sẽ nằm trong executable `$PATH`.

* Khi đó có thể dùng trực tiếp từ terminal.

# Phân tích Workflow code

## Chạy check toàn bộ module

* Có thể phân tích một package cụ thể bằng cách truyền tên package vào command.

* Cách đơn giản nhất là chạy recursive check toàn bộ code trong module.

* Chạy lệnh sau từ root directory của module:

```bash
workflowcheck ./...
```

## Kết quả khi không có lỗi

* Nếu tool không tìm thấy non-deterministic code trong `Workflow`, command sẽ exit mà không in output.

* Nghĩa là không có violation nào được phát hiện.

## Kết quả khi có lỗi

* Nếu phát hiện non-deterministic code, tool sẽ report từng violation ra standard output.

* Output được trình bày dạng hierarchical display.

* Display này giúp chỉ ra root cause của violation.

# Bỏ qua violation khi cần

## Ignore một dòng cụ thể

* Nếu chắc chắn violation được báo không phải vấn đề, có thể thêm comment đặc biệt vào cuối dòng code:

```go
//workflowcheck:ignore
```

* Lưu ý quan trọng:

  * Comment phải nằm ở cuối dòng cần ignore.
  * Không được có khoảng trắng sau `//`.

## Ý nghĩa

* Sau khi thêm comment này, `workflowcheck` sẽ bỏ qua dòng code đó trong các lần check sau.

# Cấu hình nâng cao

## Dùng configuration file

* Ngoài ignore từng dòng, `workflowcheck` còn hỗ trợ configuration file.

* Có thể dùng config file để:

  * Ignore toàn bộ usages của một function cụ thể.
  * Skip checking một số file dựa trên tên file.

* Các option khác được mô tả trong documentation của `workflowcheck`: https://pkg.go.dev/go.temporal.io/sdk/contrib/tools/workflowcheck#section-readme. 