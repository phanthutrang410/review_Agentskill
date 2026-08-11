# Deploy deck lên GitHub Pages

Repo: https://github.com/phanthutrang410/review_Agentskill

## 1. Cấu trúc phục vụ

GitHub Pages đọc từ **gốc repo**, nên bản dựng của deck nằm ở gốc:

```
review_Agentskill/
├── index.html                          ← deck đã dựng, 1.63 MB, tự chứa
├── .nojekyll                           ← tắt Jekyll, phục vụ file nguyên trạng
├── README.md
├── docs/                               ← 5 tài liệu review
└── slides/
    ├── agent_skill_review_slides.html  ← tệp nguồn (bản dùng cho artifact)
    └── DEPLOY.md                       ← tệp này
```

`index.html` là bản **dựng** từ `slides/agent_skill_review_slides.html`. Khác biệt duy nhất giữa
hai tệp là phần `<head>`: tệp nguồn không có `<!doctype>`, `<head>`, `charset`, vì runtime của
artifact tự bọc. Đẩy thẳng tệp nguồn lên host tĩnh sẽ vỡ dấu tiếng Việt khi mở file local.

## 2. Bật Pages (làm một lần)

**Settings → Pages** → mục *Build and deployment*:

- Source: **Deploy from a branch**
- Branch: **main**, thư mục **/ (root)**
- Save

Chờ 1–2 phút rồi tải lại trang, URL sẽ hiện ở đầu:

```
https://phanthutrang410.github.io/review_Agentskill/
```

Link tới một slide cụ thể — nối thêm `#/N`:

```
https://phanthutrang410.github.io/review_Agentskill/#/25
```

## 3. Những gì đã xử lý sẵn

| Hạng mục | Trạng thái |
|---|---|
| `<!doctype>`, `<head>`, `charset=utf-8` | Có trong `index.html` |
| Ảnh, CSS, JS | Nhúng inline toàn bộ — không request nào rời trang, chạy được cả khi offline |
| Hash routing `#/N` | Hoạt động đầy đủ trên host tĩnh; mở link có hash là nhảy thẳng slide, không animation |
| Theme sáng / tối | Tự theo thiết lập máy người xem |
| In ra PDF | `Ctrl+P` → mỗi slide một trang ngang |
| `noindex, nofollow` | Đã bật — xem mục 4 |

## 4. Về thẻ `noindex`

`index.html` chứa dòng:

```html
<meta name="robots" content="noindex, nofollow">
```

Ai có link vẫn xem bình thường; search engine không lập chỉ mục trang. Lý do: deck có 10 ảnh cắt
trực tiếp từ hai bản PDF arXiv (P1 Figure 1/2/3 + Table 9; P2 Figure 1/2/3/4 + Table 7/8).

Dùng trong seminar hoặc chia sẻ link có kiểm soát thì đây là trích dẫn học thuật thông thường.
Muốn để search engine index công khai thì nên kiểm tra giấy phép hai công bố trước — arXiv có bài
cấp CC-BY, có bài chỉ cấp *non-exclusive license to distribute*, hai chế độ này khác nhau về quyền
tái sử dụng hình. Xem mục **License** ở cột phải trang abstract:

- https://arxiv.org/abs/2605.30723
- https://arxiv.org/abs/2606.18837

Bỏ chặn index: xoá đúng dòng `<meta name="robots" ...>` trong `index.html`.

*Lưu ý kỹ thuật:* với project page (`user.github.io/<repo>/`), tệp `robots.txt` trong repo **không
có tác dụng** — crawler chỉ đọc `robots.txt` ở gốc tên miền `phanthutrang410.github.io`. Phải dùng
thẻ meta, không dùng `robots.txt`.

## 5. Cập nhật deck về sau

Sửa tệp nguồn `slides/agent_skill_review_slides.html`, rồi dựng lại `index.html`. Chạy từ gốc repo:

```powershell
$src = ".\slides\agent_skill_review_slides.html"
$dst = ".\index.html"
$body = [System.IO.File]::ReadAllText($src)
$m = [regex]::Match($body, '^\s*<title>(?<t>.*?)</title>\s*', 'Singleline')
$title = $m.Groups['t'].Value
$body = $body.Substring($m.Length)
$head = @"
<!doctype html>
<html lang="vi">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="robots" content="noindex, nofollow">
<meta name="color-scheme" content="light dark">
<title>$title</title>
<style>
html{-webkit-text-size-adjust:100%}
body{margin:0}
img{max-width:100%;height:auto;border:0}
button{font:inherit;color:inherit}
table{border-collapse:collapse}
</style>
</head>
<body>
"@
[System.IO.File]::WriteAllText($dst, $head + $body + "`n</body>`n</html>`n", (New-Object System.Text.UTF8Encoding($false)))
```

Rồi `git add -A; git commit -m "cap nhat deck"; git push`.

**Không** dùng kiểu đọc-và-ghi cùng một tệp trong một biểu thức — mở tệp ở chế độ ghi sẽ cắt tệp về
rỗng trước khi kịp đọc. Đoạn trên đọc vào biến trước, ghi ra tệp đích khác, nên an toàn.

## 6. Custom domain (tuỳ chọn)

1. Ở nhà cung cấp DNS, thêm bản ghi `CNAME` trỏ subdomain về `phanthutrang410.github.io`.
2. **Settings → Pages → Custom domain**, nhập tên miền, Save.
3. Chờ DNS check xong rồi tích **Enforce HTTPS**.
