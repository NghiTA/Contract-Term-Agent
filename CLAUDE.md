# Contract Term Management Agent

Trợ lý AI giúp **Team Legal** quản lý vòng đời hợp đồng và cảnh báo cho **Team Business** trước khi hợp đồng hết hiệu lực.

## Vấn đề cần giải quyết
Team Legal quản lý nhiều hợp đồng và cần biết khi nào một hợp đồng hết hạn để cảnh báo team Business, tránh tình trạng hợp đồng mất hiệu lực mà cả Công ty lẫn Đối tác đều không biết.

## Input
- Một **folder hợp đồng** (`CONTRACTS_DIR`). Mỗi hợp đồng là **một thư mục con**, bên trong gồm hợp đồng gốc và (nếu có) các **văn bản/phụ lục ghi nhận việc gia hạn**.
- Hoặc nội dung văn bản gửi trực tiếp qua API `/analyze-text`.
- Định dạng hỗ trợ: `.pdf`, `.docx`, `.txt`, `.md`.

## Nhiệm vụ của Agent
1. **Xác định thời hạn** hợp đồng → tính và đưa ra **ngày hết hiệu lực**.
2. **Cảnh báo trước 15 ngày** kể từ thời điểm hết hạn, phân biệt 2 trường hợp:
   - (i) Hết hạn **và tự động gia hạn** → nhắc xác nhận có muốn dừng/không gia hạn, theo dõi thời hạn mới.
   - (ii) Hết hạn **và KHÔNG tự động gia hạn** → nhắc chủ động ký phụ lục/hợp đồng mới.
3. **Tiếp tục theo dõi** dựa trên văn bản gia hạn **mới nhất**: khi có phụ lục mới, lấy thời hạn theo văn bản mới nhất có hiệu lực.

## RULE CỨNG (bắt buộc tuân thủ)
- **KHÔNG tự ý suy diễn.** Chỉ dùng nội dung có thật trong hợp đồng.
- Đảm bảo đúng nội dung hợp đồng; không bịa số liệu, ngày tháng, điều khoản.
- **Trường hợp không xác định được → phản hồi lại ngay** với trạng thái `KHONG_XAC_DINH` và nêu rõ lý do để con người kiểm tra. Không đoán.

## Trạng thái trả về
- `CON_HIEU_LUC` – còn xa ngày hết hạn.
- `SAP_HET_HAN` – trong vòng 15 ngày → **cần cảnh báo**.
- `DA_HET_HAN` – đã quá hạn.
- `KHONG_XAC_DINH` – thiếu dữ liệu/không chắc chắn → cần người kiểm tra.

Loại cảnh báo: `TU_DONG_GIA_HAN`, `KHONG_TU_DONG_GIA_HAN`, `GIA_HAN_KHONG_RO`.

## Kiến trúc kỹ thuật
- **FastAPI** web service (deploy lên GreenNode AgentBase Runtime). Có `/health` trả 200.
- Trích xuất thông tin hợp đồng bằng LLM qua **GreenNode MaaS** (OpenAI-compatible API), mặc định model **Minimax**, `temperature=0` để giảm bịa.
- Logic cảnh báo (ngày hết hạn, cửa sổ 15 ngày, phân loại gia hạn) được tính **bằng code Python** chứ không giao cho model, để đảm bảo chính xác và tuân thủ rule cứng.

## Biến môi trường
| Biến | Mặc định | Ý nghĩa |
|------|----------|---------|
| `MAAS_API_KEY` | *(bắt buộc)* | API key MaaS từ GreenNode Portal |
| `MAAS_BASE_URL` | `https://maas.aiplatform.vngcloud.vn/v1` | Endpoint OpenAI-compatible |
| `MAAS_MODEL` | `minimax` | Model dùng để trích xuất (Gemma/Qwen/Minimax) |
| `CONTRACTS_DIR` | `./contracts` | Folder chứa hợp đồng |
| `ALERT_WINDOW_DAYS` | `15` | Số ngày cảnh báo trước hạn |
| `PORT` | `8080` | Cổng web service |

## Endpoints
- `GET /health` – kiểm tra sức khỏe (AgentBase yêu cầu).
- `GET /scan` – quét cả folder, trả cảnh báo cho từng hợp đồng.
- `GET /analyze?name=<tên>` – phân tích 1 hợp đồng.
- `POST /analyze-text` – phân tích từ nội dung text trực tiếp.
- `GET /` – giao diện web demo. `GET /docs` – Swagger UI.
