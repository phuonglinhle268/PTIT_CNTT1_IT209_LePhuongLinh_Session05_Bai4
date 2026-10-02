# Phân Tích Thực Hành: Quy Trình Phân Nhánh Hotfix & Gitflow Thực Tế

## 1. Phân tích mô hình phân nhánh Gitflow và Quy trình xử lý Hotfix

### 1.1. Vai trò của các nhánh trong mô hình Gitflow
- **Nhánh `main` (Production):** Lưu trữ mã nguồn phiên bản ổn định (stable release) đang chạy thực tế trên môi trường người dùng cuối. Mọi thay đổi trên `main` đều gắn liền với các mốc phát hành chính thức qua `git tag`.
- **Nhánh `develop` (Development):** Nhánh tích hợp chính cho các tính năng mới trong tương lai. Tại thời điểm phát sinh lỗi khẩn cấp, `develop` đang chứa các tính năng dở dang (Work In Progress) nên không thể đưa lên production được.
- **Nhánh `hotfix/*` (Bản vá nóng khẩn cấp):** Nhánh ngắn hạn (temporary branch) được rẽ nhánh trực tiếp từ `main` nhằm cô lập và xử lý sự cố nghiêm trọng mà không làm ảnh hưởng đến tiến độ phát triển của nhánh `develop`.

### 1.2. Nguyên lý đồng bộ hai chiều (Dual-Merge) trong xử lý Hotfix
1. **Cô lập lỗi (Branching from `main`):** Tách nhánh `hotfix/v1.0.1` từ `main` (v1.0.0) đảm bảo bản vá chỉ chứa duy nhất phần sửa lỗi, không bị lẫn các đoạn code chưa hoàn thiện từ `develop`.
2. **Triển khai Production (`hotfix` -> `main`):** Gộp `hotfix/v1.0.1` vào `main` bằng `--no-ff` (No Fast-Forward) để lưu lại vết merge commit, sau đó gắn tag phiên bản `v1.0.1` để lập tức release ra môi trường production.
3. **Chống trôi lỗi (`hotfix` -> `develop`):** Bắt buộc gộp ngược `hotfix/v1.0.1` vào `develop`. Thao tác này đảm bảo toàn bộ mã nguồn đang phát triển kế thừa bản vá bảo mật, triệt tiêu hoàn toàn nguy cơ tái phát sinh lỗi (Bug Regression) khi phát hành các phiên bản lớn tiếp theo.
4. **Giải quyết xung đột (Conflict Resolution):** Khi merge vào `develop`, nếu có xung đột xảy ra tại các dòng code đang sửa dở, tiến hành kết hợp cả bản vá bảo mật và mã nguồn tính năng mới một cách chuẩn xác trước khi commit.
5. **Dọn dẹp (Cleanup):** Sau khi cả `main` và `develop` đều đã nhận được bản vá, xóa nhánh `hotfix/v1.0.1` cục bộ để giữ cấu trúc kho chứa gọn gàng.

---

## 2. Sơ đồ phân nhánh và Đồ thị Commit (ASCII Diagram)

### 2.1. Sơ đồ nguyên lý quy trình Gitflow Hotfix
```text
(Tag: v1.0.0)
[main]    --- [e54c4f4] --------------------------------- [fe4741b] (Tag: v1.0.1 - Release Production)
                \                                         /
[hotfix]         \-------- [b941fc4: Security Patch] ----/ (Xóa nhánh sau khi merge xong)
                  \                                       \
[develop]          \--- [36a884c: Feature in progress] ----\ --- [92d0c55: Resolved & Synced]
```

### 2.2. Đồ thị Commit thực tế của Repository (`git log --graph`)
```text
*   92d0c55 (HEAD -> develop) merge: dong bo ban va hotfix/v1.0.1 vao develop
|\  
* | 36a884c feat(develop): dang phat trien
| | * fe4741b (tag: v1.0.1, main) merge: gop hotfix/v1.0.1 vao main
| |/| 
|/|/  
| * b941fc4 fix(security): va loi ro ri du lieu nguoi dung
|/    
* e54c4f4 (tag: v1.0.0) feat: release version 1.0.0
```

---

## 3. Các bước thực hiện chi tiết & Nhật ký thực thi (Terminal Logs)

### Bước 1: Khởi tạo kho chứa, phát hành phiên bản `v1.0.0` trên `main`
Tạo dự án, tạo file `app.js` phiên bản `v1.0.0` và gắn tag release ban đầu:
```powershell
PS C:\Users\plinh> cd "D:\DevOps - RA\Session05"
PS D:\DevOps - RA\Session05> mkdir ss05-bai4


    Directory: D:\DevOps - RA\Session05


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         10/2/2026   1:49 PM                ss05-bai4


PS D:\DevOps - RA\Session05> cd ss05-bai4
PS D:\DevOps - RA\Session05\ss05-bai4> git init -b main
Initialized empty Git repository in D:/DevOps - RA/Session05/ss05-bai4/.git/
PS D:\DevOps - RA\Session05\ss05-bai4> echo "console.log('Production v1.0.0 running');" > app.js
PS D:\DevOps - RA\Session05\ss05-bai4> git add app.js
PS D:\DevOps - RA\Session05\ss05-bai4> git commit -m "feat: release version 1.0.0"
[main (root-commit) e54c4f4] feat: release version 1.0.0
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 app.js
PS D:\DevOps - RA\Session05\ss05-bai4> git tag -a v1.0.0 -m "Release version 1.0.0"
```

---

### Bước 2: Tạo nhánh `develop` và phát triển tính năng mới
Tách nhánh `develop` từ `main` và commit tính năng đang phát triển dở dang:
```powershell
PS D:\DevOps - RA\Session05\ss05-bai4> git checkout -b develop
Switched to a new branch 'develop'
PS D:\DevOps - RA\Session05\ss05-bai4> echo "console.log('Feature user profile in progress');" >> app.js
PS D:\DevOps - RA\Session05\ss05-bai4> git add app.js
PS D:\DevOps - RA\Session05\ss05-bai4> git commit -m "feat(develop): dang phat trien"
[develop 36a884c] feat(develop): dang phat trien
 1 file changed, 0 insertions(+), 0 deletions(-)
```

---

### Bước 3: Tạo nhánh `hotfix/v1.0.1` từ `main` và thực hiện vá lỗi bảo mật
Khi phát hiện sự cố rò rỉ dữ liệu, quay lại `main` để tách nhánh hotfix khẩn cấp và commit bản vá:
```powershell
PS D:\DevOps - RA\Session05\ss05-bai4> git checkout main
Switched to branch 'main'
PS D:\DevOps - RA\Session05\ss05-bai4> git checkout -b hotfix/v1.0.1
Switched to a new branch 'hotfix/v1.0.1'
PS D:\DevOps - RA\Session05\ss05-bai4> echo "// [SECURITY PATCH] Da an thong tin nhay cam cua user" >> app.js
PS D:\DevOps - RA\Session05\ss05-bai4> git add app.js
PS D:\DevOps - RA\Session05\ss05-bai4> git commit -m "fix(security): va loi ro ri du lieu nguoi dung"
[hotfix/v1.0.1 b941fc4] fix(security): va loi ro ri du lieu nguoi dung
 1 file changed, 0 insertions(+), 0 deletions(-)
```

---

### Bước 4: Gộp `hotfix/v1.0.1` vào `main` và gắn tag phát hành `v1.0.1`
Chuyển về `main`, gộp nhánh hotfix bằng `--no-ff` và tạo annotated tag `v1.0.1`:
```powershell
PS D:\DevOps - RA\Session05\ss05-bai4> git checkout main
Switched to branch 'main'
PS D:\DevOps - RA\Session05\ss05-bai4> git merge --no-ff hotfix/v1.0.1 -m "merge: gop hotfix/v1.0.1 vao main"
Merge made by the 'ort' strategy.
 app.js | Bin 88 -> 198 bytes
 1 file changed, 0 insertions(+), 0 deletions(-)
PS D:\DevOps - RA\Session05\ss05-bai4> git tag -a v1.0.1 -m "Release Hotfix 1.0.1"
```

---

### Bước 5: Gộp `hotfix/v1.0.1` vào `develop`, xử lý Conflict và dọn dẹp nhánh
Chuyển về `develop`, gộp bản vá để đồng bộ, giải quyết xung đột mã nguồn và xóa nhánh hotfix đã hoàn thành:
```powershell
PS D:\DevOps - RA\Session05\ss05-bai4> git checkout develop
Switched to branch 'develop'
PS D:\DevOps - RA\Session05\ss05-bai4> git merge --no-ff hotfix/v1.0.1 -m "merge: dong bo ban va hotfix/v1.0.1 vao develop"
warning: Cannot merge binary files: app.js (HEAD vs. hotfix/v1.0.1)
Auto-merging app.js
CONFLICT (content): Merge conflict in app.js
Automatic merge failed; fix conflicts and then commit the result.

PS D:\DevOps - RA\Session05\ss05-bai4> Set-Content -Path app.js -Value "console.log('Production v1.0.0 running');`n// [SECURITY PATCH] Da an thong tin nhay cam cua user`nconsole.log('Feature user profile in progress');" -Encoding utf8
PS D:\DevOps - RA\Session05\ss05-bai4> git add app.js
warning: in the working copy of 'app.js', LF will be replaced by CRLF the next time Git touches it
PS D:\DevOps - RA\Session05\ss05-bai4> git commit -m "merge: dong bo ban va hotfix/v1.0.1 vao develop"
[develop 92d0c55] merge: dong bo ban va hotfix/v1.0.1 vao develop

PS D:\DevOps - RA\Session05\ss05-bai4> git branch -d hotfix/v1.0.1
Deleted branch hotfix/v1.0.1 (was b941fc4).
```

---

## 4. Kết quả kiểm tra và Xác minh trạng thái

### 4.1. Kiểm tra danh sách nhánh hiện hữu (`git branch -a`)
Nhánh `hotfix/v1.0.1` đã được dọn dẹp an toàn, chỉ còn 2 nhánh dài hạn `main` và `develop`:
```powershell
PS D:\DevOps - RA\Session05\ss05-bai4> git branch -a
* develop
  main
```

### 4.2. Kiểm tra danh sách phiên bản phát hành (`git tag`)
Hệ thống ghi nhận đầy đủ 2 phiên bản release: bản ban đầu `v1.0.0` và bản vá lỗi khẩn cấp `v1.0.1`:
```powershell
PS D:\DevOps - RA\Session05\ss05-bai4> git tag
v1.0.0
v1.0.1
```

### 4.3. Kiểm tra đồ thị lịch sử gộp nhánh toàn cục (`git log --graph --oneline --all`)
Lịch sử phản ánh chính xác quy trình Gitflow chuẩn: commit `b941fc4` của hotfix được rẽ nhánh từ `e54c4f4` (v1.0.0) và được merge song song vào cả `main` (`fe4741b` mang tag `v1.0.1`) lẫn `develop` (`92d0c55`):
```powershell
PS D:\DevOps - RA\Session05\ss05-bai4> git log --graph --oneline --all
*   92d0c55 (HEAD -> develop) merge: dong bo ban va hotfix/v1.0.1 vao develop
|\  
* | 36a884c feat(develop): dang phat trien
| | * fe4741b (tag: v1.0.1, main) merge: gop hotfix/v1.0.1 vao main
| |/| 
|/|/  
| * b941fc4 fix(security): va loi ro ri du lieu nguoi dung
|/    
* e54c4f4 (tag: v1.0.0) feat: release version 1.0.0
```
