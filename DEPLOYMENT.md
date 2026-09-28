
# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Đỗ Mạnh Đoan
| Mã học viên | 2A202602839 |
| Repo | https://github.com/mDoanzz43/K4-L3A-DAY12-DoManhDoan-2A202602839-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-04f4.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis service nội bộ của Railway |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Các lệnh dưới đây dành cho Windows PowerShell 5.1. API key được đọc từ `.env`
cục bộ và không được in ra màn hình.

```powershell
$URL = "https://agent-production-04f4.up.railway.app"
$keyLine = Get-Content -LiteralPath ".env" |
    Where-Object { $_ -match '^\s*AGENT_API_KEY\s*=' } |
    Select-Object -Last 1
$env:AGENT_API_KEY = ($keyLine -split "=", 2)[1].Trim()

$json = @{ question = "Deploy là gì?" } | ConvertTo-Json -Compress
$bodyBytes = [System.Text.Encoding]::UTF8.GetBytes($json)

# 1. Liveness — mong đợi 200
Invoke-WebRequest -Uri "$URL/health" -UseBasicParsing

# 2. Readiness — mong đợi 200 và redis=true
Invoke-WebRequest -Uri "$URL/ready" -UseBasicParsing

# 3. Không có API key — mong đợi 401
try {
    Invoke-WebRequest -Method POST -Uri "$URL/ask" `
        -ContentType "application/json; charset=utf-8" `
        -Body $bodyBytes -UseBasicParsing
} catch {
    [int]$_.Exception.Response.StatusCode
}

# 4. Có API key — mong đợi 200 kèm câu trả lời
$headers = @{
    "X-API-Key" = $env:AGENT_API_KEY
    "X-User-Id" = "sv-test"
}
$response = Invoke-WebRequest -Method POST -Uri "$URL/ask" `
    -Headers $headers -ContentType "application/json; charset=utf-8" `
    -Body $bodyBytes -UseBasicParsing
$response.StatusCode
$response.RawContentStream.Position = 0
[System.Text.Encoding]::UTF8.GetString($response.RawContentStream.ToArray())

# 5. Rate limit — request 1–10 trả 200, request 11–15 trả 429
$rateHeaders = @{
    "X-API-Key" = $env:AGENT_API_KEY
    "X-User-Id" = "sv-rate-" + [guid]::NewGuid().ToString("N")
}
1..15 | ForEach-Object {
    try {
        $result = Invoke-WebRequest -Method POST -Uri "$URL/ask" `
            -Headers $rateHeaders -ContentType "application/json; charset=utf-8" `
            -Body $bodyBytes -UseBasicParsing
        "Request $_`: $($result.StatusCode)"
    } catch {
        "Request $_`: $([int]$_.Exception.Response.StatusCode)"
    }
}
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
GET /health
HTTP 200
{"status":"ok","service":"day12-agent","version":"1.0.0"}

GET /ready
HTTP 200
{"status":"ready","redis":true}

POST /ask không có X-API-Key
HTTP 401

POST /ask có X-API-Key hợp lệ
HTTP 200, response có trường answer

Rate-limit test với 15 request cùng X-User-Id
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

## Phương Án Dự Phòng

Không sử dụng phương án local fallback; service đang chạy thật trên Railway.
