# Bài 3: Xử lý xung đột phức tạp trong quá trình Rebase

## Mục tiêu
- Phân biệt cách xử lý xung đột khi **Merge** và khi **Rebase**.
- Giải quyết xung đột **từng chặng** (step-by-step) trong tiến trình rebase.
- Đưa các commit của `feature-api` nối tiếp lên đầu `main`, tạo lịch sử **thẳng hàng**, không có merge commit.

## Môi trường
- Windows 10, Git Bash (MINGW64)
- Thư mục thực hành: `/d/rikkei/IT209/Ss5/Ls3`
- Trình soạn thảo: Notepad

---

## Chuẩn bị kịch bản

### Commit gốc trên `main`

```bash
mkdir -p /d/rikkei/IT209/Ss5/Ls3 && cd /d/rikkei/IT209/Ss5/Ls3
git init -b main
git config --local user.name "thang-260704"
git config --local user.email "phanhtran0501@gmail.com"
git config --local core.editor notepad
printf '{\n  "port": 8080,\n  "debug": false\n}\n' > config.json
git add config.json && git commit -m "init config"
```

### Nhánh `feature-api` – 2 commit

```bash
git checkout -b feature-api
printf '{\n  "port": 9000,\n  "debug": false\n}\n' > config.json
git commit -am "feat: change port"
printf '{\n  "port": 9000,\n  "debug": true\n}\n' > config.json
git commit -am "feat: enable debug"
```

### Nhánh `main` – 2 commit mới của thành viên khác

```bash
git checkout main
printf '{\n  "port": 8081,\n  "debug": false\n}\n' > config.json
git commit -am "update port on main"
printf '{\n  "port": 8081,\n  "debug": false,\n  "env": "production"\n}\n' > config.json
git commit -am "add env config"
```

Đồ thị trước khi rebase. Hai nhánh rẽ nhau từ `init config`:

```text
* c4829ac (HEAD -> main) add env config
* e1fc125 update port on main
| * a8ac2d8 (feature-api) feat: enable debug
| * 96e6abf feat: change port
|/
* 0d6adeb init config
```

![Trước khi rebase](images/01-before-rebase.png)

---

## Tiến hành Rebase

```bash
git checkout feature-api
git rebase main
```

Git lấy lần lượt từng commit của `feature-api`, rồi **áp dụng lại (replay)** lên đỉnh của `main`. Mỗi commit đụng tới dòng mà `main` đã sửa thì rebase dừng lại ở chặng đó.

### ⚔️ Chặng 1 – Xung đột ở commit `feat: change port`

```text
CONFLICT (content): Merge conflict in config.json
error: could not apply 96e6abf... feat: change port
```

Dấu nhắc chuyển thành `(feature-api|REBASE 1/2)`, nghĩa là đang ở chặng 1 trên tổng 2 commit. `git status` báo `interactive rebase in progress; onto c4829ac`, và `config.json` nằm trong mục `both modified`.

Nội dung file lúc xung đột:

```text
{
<<<<<<< HEAD
  "port": 8081,
  "debug": false,
  "env": "production"
=======
  "port": 9000,
  "debug": false
>>>>>>> 96e6abf (feat: change port)
}
```

**Phân tích:**
- Phía `HEAD` là **code của `main`** (port `8081`, có thêm `env`).
- Phía dưới `=======` là thay đổi của commit `feat: change port` (port `9000`).

**Cách giải quyết:** lấy port `9000` theo mục đích của commit tính năng, đồng thời **giữ nguyên `"env": "production"` của `main`** để không phá code nhánh chính.

```json
{
  "port": 9000,
  "debug": false,
  "env": "production"
}
```

```bash
notepad config.json          # sửa thủ công, xoá các dấu <<<<<<< ======= >>>>>>>
git add config.json
GIT_EDITOR=true git rebase --continue
```

`GIT_EDITOR=true` giữ nguyên thông điệp commit cũ mà không mở trình soạn thảo.

![Xung đột chặng 1](images/02-conflict-1.png)

### ⚔️ Chặng 2 – Xung đột ở commit `feat: enable debug`

Rebase tiếp tục với commit thứ hai và lại dừng:

```text
CONFLICT (content): Merge conflict in config.json
error: could not apply a8ac2d8... feat: enable debug
```

Dấu nhắc lúc này là `(feature-api|REBASE 2/2)`. Commit `feat: change port` đã được tạo lại với mã mới `b18d171`.

```text
{
  "port": 9000,
<<<<<<< HEAD
  "debug": false,
  "env": "production"
}
=======
  "debug": true
}
>>>>>>> a8ac2d8 (feat: enable debug)
```

**Phân tích:**
- `HEAD` lúc này là kết quả đã giải quyết ở chặng 1: port 9000, debug false, có env.
- Commit `feat: enable debug` muốn đổi `debug` thành `true`, nhưng ở phiên bản gốc của nó dòng `debug` không có dấu phẩy và chưa có dòng `env`, nên Git không tự ghép được.

**Cách giải quyết:** bật `debug: true` theo commit tính năng và vẫn giữ `env` của `main`:

```json
{
  "port": 9000,
  "debug": true,
  "env": "production"
}
```

```bash
notepad config.json
git add config.json
GIT_EDITOR=true git rebase --continue
```

```text
Successfully rebased and updated refs/heads/feature-api.
```

![Xung đột chặng 2](images/03-conflict-2.png)

---

## Kết quả

```bash
git log --graph --oneline --all
cat config.json
```

```text
* 4274968 (HEAD -> feature-api) feat: enable debug
* b18d171 feat: change port
* c4829ac (main) add env config
* e1fc125 update port on main
* 0d6adeb init config

{
  "port": 9000,
  "debug": true,
  "env": "production"
}
```

![Lịch sử thẳng hàng sau rebase](images/04-final-log.png)

- Lịch sử là **một đường thẳng**, không có nhánh rẽ (`|\`, `|/`) và **không có merge commit**.
- Hai commit của `feature-api` nằm **ngay sau** commit mới nhất của `main` (`add env config`).
- Hai commit tính năng mang **mã hash mới**: `96e6abf` thành `b18d171`, `a8ac2d8` thành `4274968`. Lý do là rebase tạo lại commit trên nền mới `c4829ac`.
- `config.json` giữ đủ thay đổi của cả hai phía: `env` từ `main`, `port` và `debug` từ `feature-api`.

---

## So sánh xử lý xung đột: Merge và Rebase

| Tiêu chí | Merge | Rebase |
|---|---|---|
| Số lần giải quyết | **Một lần** cho toàn bộ thay đổi | **Từng chặng**, mỗi commit có thể conflict riêng |
| Lệnh tiếp tục | `git add` + `git commit` | `git add` + `git rebase --continue` |
| Huỷ giữa chừng | `git merge --abort` | `git rebase --abort` |
| Bỏ qua commit đang lỗi | Không có | `git rebase --skip` |
| `HEAD` / "ours" là | Nhánh **hiện tại** đang đứng | Nhánh **đích** (`main`) cùng các commit đã replay |
| "theirs" là | Nhánh được gộp vào | Commit **của mình** đang được replay |
| Lịch sử sau cùng | Có merge commit, đồ thị rẽ nhánh | **Thẳng hàng**, không có merge commit |

> ⚠️ Khi rebase, ý nghĩa `ours` và `theirs` **đảo ngược** so với merge. Phía `HEAD` là code của `main`, còn code của mình nằm ở phía dưới `=======`. Đây là điểm dễ nhầm nhất khi giải quyết xung đột lúc rebase.

## Đối chiếu yêu cầu

| Yêu cầu | Kết quả |
|---|---|
| `main` có 2 commit mới sửa `config.json` | ✅ `update port on main`, `add env config` |
| `feature-api` có 2 commit sửa cùng các dòng | ✅ `feat: change port`, `feat: enable debug` |
| Rebase phát sinh xung đột ở từng commit | ✅ 2 chặng xung đột |
| Giải quyết thủ công từng chặng, không dùng merge | ✅ `git add` + `git rebase --continue` |
| Không phá code của `main` (giữ `env`) | ✅ |
| `git log --graph --oneline` thẳng hàng, không có merge commit | ✅ |
