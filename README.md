# 🛡️ Contract Term Management Agent

Trợ lý AI giúp **Team Legal** quản lý vòng đời hợp đồng: tự động xác định ngày hết hiệu lực và **cảnh báo trước 15 ngày**, phân biệt hợp đồng có/không tự động gia hạn. Xây dựng cho **GreenNode Claw-a-thon 2026**, triển khai trên **AgentBase**.

## Tính năng
- 📅 Trích xuất ngày hiệu lực, thời hạn → tính **ngày hết hiệu lực**.
- 🔔 Cảnh báo trước **15 ngày** theo 2 trường hợp: tự động gia hạn / không tự động gia hạn.
- 📎 Theo dõi theo **văn bản gia hạn mới nhất** (phụ lục).
- 🚫 **Rule cứng:** không suy diễn; không xác định được thì trả `KHONG_XAC_DINH` ngay.
- 🌐 Web API (FastAPI) + giao diện demo, có `/health` cho AgentBase.

## Cấu trúc dữ liệu hợp đồng
```
contracts/
├── HD-2024-001/                # 1 hợp đồng = 1 thư mục
│   ├── hop-dong-goc.pdf
│   └── phu-luc-gia-han-01.docx # văn bản gia hạn (mới hơn → ưu tiên)
└── HD-2025-007/
    └── hop-dong.docx
```
> Cũng có thể để file phẳng: mỗi file = 1 hợp đồng không có văn bản gia hạn.

## Chạy local
```bash
pip install -r requirements.txt
export MAAS_API_KEY="<api-key-tu-greennode-portal>"
export MAAS_MODEL="minimax"      # hoặc gemma / qwen
python agent.py                  # mở http://localhost:8080
```

## Biến môi trường
| Biến | Mặc định | Ý nghĩa |
|------|----------|---------|
| `MAAS_API_KEY` | *(bắt buộc)* | API key MaaS (GreenNode Portal → MaaS API Keys) |
| `MAAS_BASE_URL` | `https://maas.aiplatform.vngcloud.vn/v1` | Endpoint OpenAI-compatible |
| `MAAS_MODEL` | `minimax` | Model trích xuất |
| `CONTRACTS_DIR` | `./contracts` | Folder hợp đồng |
| `ALERT_WINDOW_DAYS` | `15` | Số ngày cảnh báo trước hạn |
| `PORT` | `8080` | Cổng dịch vụ |

> ⚠️ Kiểm tra lại `MAAS_BASE_URL` và tên model theo đúng thông tin trên GreenNode Portal của team (mục MaaS). Cập nhật nếu khác.

## API
| Method | Path | Mô tả |
|--------|------|-------|
| GET | `/health` | Health check → `{"status":"ok"}` |
| GET | `/scan?folder=&today=` | Quét cả folder, trả cảnh báo từng hợp đồng |
| GET | `/analyze?name=&folder=&today=` | Phân tích 1 hợp đồng |
| POST | `/analyze-text` | Phân tích từ nội dung text |
| GET | `/` , `/docs` | Giao diện demo, Swagger |

Ví dụ:
```bash
curl http://localhost:8080/health
curl "http://localhost:8080/scan"
curl -X POST http://localhost:8080/analyze-text \
  -H "Content-Type: application/json" \
  -d '{"contract_name":"HD demo","documents":["Hợp đồng có hiệu lực 01/01/2025, thời hạn 12 tháng, không tự động gia hạn."]}'
```

## Docker
```bash
docker build -t contract-term-agent .
docker run -p 8080:8080 -e MAAS_API_KEY="<key>" -e MAAS_MODEL="minimax" contract-term-agent
```

## Deploy lên AgentBase (tóm tắt theo Demo Checklist)
1. Lấy `MAAS_API_KEY`, `Client ID`, `Client Secret` từ GreenNode Portal.
2. Đưa bộ skill **AgentBase** (`git clone github.com/vngcloud/greennode-agentbase-skills`) vào **cùng folder** với agent này.
3. Prompt: *"Sử dụng skill này để deploy agent của tôi lên AgentBase"* → nhập credential, chọn model & runtime size (2x4 hoặc 4x4).
4. Kiểm tra Runtime status = **ACTIVE**, verify `endpoint/health` trả **200**.
5. Prompt **"Chuyển endpoint sang public"** để có link chia sẻ; rồi **push lên GitHub (PUBLIC)**.

## Lưu ý thiết kế
Việc tính ngày hết hạn, cửa sổ 15 ngày và phân loại gia hạn được thực hiện **bằng code** (không giao cho model) để đảm bảo chính xác và tuân thủ rule cứng. Model chỉ làm nhiệm vụ trích xuất thông tin từ văn bản với `temperature=0`.
