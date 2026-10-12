# Thiết kế RESTful API: Quản lý Task và User

## 1. Mô hình dữ liệu

### User

| Trường | Kiểu | Mô tả | Ràng buộc |
|---|---|---|---|
| id | long | Mã người dùng | Tự sinh |
| name | string | Tên người dùng | Bắt buộc, không rỗng |
| email | string | Email | Bắt buộc, đúng định dạng email |
| role | string | Vai trò: `admin`, `member` | Bắt buộc |

### Task

| Trường | Kiểu | Mô tả | Ràng buộc |
|---|---|---|---|
| id | long | Mã công việc | Tự sinh |
| title | string | Tiêu đề | Bắt buộc, không rỗng |
| status | string | Trạng thái: `todo`, `in_progress`, `done` | Bắt buộc, mặc định `todo` |
| priority | string | Mức ưu tiên: `low`, `medium`, `high` | Bắt buộc |
| userId | long | Khóa ngoại tới `User.id` (người được giao) | Có thể null khi chưa giao; nếu có thì User phải tồn tại |

## 2. Chi tiết từng Endpoint

### 2.1. Lấy danh sách

**GET `/api/tasks`** và **GET `/api/users`**

- Request body: không có
- Response `200 OK`:

```json
[
  { "id": 1, "title": "Viết tài liệu", "status": "todo", "priority": "high", "userId": 1 },
  { "id": 2, "title": "Sửa lỗi đăng nhập", "status": "in_progress", "priority": "medium", "userId": 2 }
]
```

### 2.2. Tạo mới

**POST `/api/tasks`**

- Request body (dùng `@RequestBody`):

```json
{ "title": "Viết tài liệu", "priority": "high", "userId": 1 }
```

- Response `201 Created`:

```json
{ "id": 3, "title": "Viết tài liệu", "status": "todo", "priority": "high", "userId": 1 }
```

- Lỗi `400 Bad Request` khi `title` rỗng, `priority` không hợp lệ hoặc `userId` không tồn tại.

**POST `/api/users`**

- Request body:

```json
{ "name": "Nguyễn Văn A", "email": "a@example.com", "role": "member" }
```

- Response `201 Created`:

```json
{ "id": 3, "name": "Nguyễn Văn A", "email": "a@example.com", "role": "member" }
```

- Lỗi `400 Bad Request` khi `name` rỗng, `email` sai định dạng hoặc `role` không hợp lệ.

### 2.3. Cập nhật một phần (PATCH)

Dùng PATCH vì chỉ sửa một trường, không gửi lại toàn bộ đối tượng.

**PATCH `/api/tasks/{id}`** (cập nhật trạng thái)

```json
{ "status": "done" }
```

- Response `200 OK`: trả về Task sau khi cập nhật
- Lỗi `400` nếu `status` không thuộc `todo`, `in_progress`, `done`
- Lỗi `404 Not Found` nếu không có Task với `id` đó

**PATCH `/api/users/{id}`** (cập nhật vai trò)

```json
{ "role": "admin" }
```

- Response `200 OK`: trả về User sau khi cập nhật
- Lỗi `400` nếu `role` không hợp lệ, `404` nếu không có User

### 2.4. Xóa

**DELETE `/api/tasks/{id}`** và **DELETE `/api/users/{id}`**

- Request body: không có
- Response `204 No Content`
- Lỗi `404 Not Found` nếu không tồn tại
- Khi xóa User đang có Task: cần chọn một cách xử lý, ví dụ trả `409 Conflict` hoặc đặt `userId` của các Task đó về null

### 2.5. Tìm kiếm và lọc

Dùng **query parameter** để lọc trên cùng tài nguyên `/api/tasks`.

**GET `/api/tasks?priority=high`**: các công việc ưu tiên cao

**GET `/api/tasks?priority=high&userId=1`**: các công việc ưu tiên cao của user có id = 1

- Response `200 OK`: mảng Task thỏa điều kiện (rỗng `[]` nếu không có)

### 2.6. Công việc của một người dùng

**GET `/api/users/{id}/tasks`**

- Biểu diễn quan hệ "User có nhiều Task" bằng đường dẫn lồng nhau
- Response `200 OK`: mảng Task có `userId = {id}`
- Lỗi `404 Not Found` nếu User không tồn tại

### 2.7. Gắn công việc cho người dùng

**PUT `/api/tasks/{id}/assignee`**

```json
{ "userId": 1 }
```

- Response `200 OK`: trả về Task với `userId` mới
- Lỗi `404 Not Found` nếu Task hoặc User không tồn tại
- Dùng PUT vì thao tác này thay thế hoàn toàn người được giao và gọi lại nhiều lần vẫn cho cùng kết quả (idempotent)
