# ChamViet - Cô Giáo Kể Chuyện Cổ Tích

Dự án **ChamViet** là một ứng dụng Voice Bot tương tác thông minh dành cho trẻ em, đóng vai trò là một "Cô giáo kể chuyện" thân thiện, ấm áp và kiên nhẫn. Cô giáo sẽ kể các câu chuyện cổ tích Việt Nam (như chuyện Lạc Long Quân và Âu Cơ), sau đó trò chuyện, đặt câu hỏi, lắng nghe và giải đáp thắc mắc của bé để giúp bé hiểu sâu hơn về bài học và câu chuyện.

---

## 🛠️ Kiến Trúc Hệ Thống

Dự án được xây dựng theo mô hình **Client-Server** sử dụng Python, FastAPI và các mô hình AI tiên tiến:

```
                      ┌────────────────────────────────────────┐
                      │                CLIENT                  │
                      │               (test.py)                │
                      └───────────────────┬────────────────────┘
                                          │ (API: Audio / Text)
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                       BACKEND SERVER                                    │
│                                         (main.py)                                       │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                     SERVICES FOLDER                                     │
├──────────────────────┬──────────────────────┬──────────────────┬────────────────────────┤
│   content_service    │     llm_service      │ session_service  │      stt_service       │
│  (Quản lý Prompt &   │ (Hỏi đáp & Phân loại │ (Quản lý Lịch sử │  (Nhận dạng giọng nói   │
│       Nội dung)      │        Ý định)       │    Trò chuyện)   │     Groq Whisper)      │
└──────────┬───────────┴──────────┬───────────┴────────┬─────────┴──────────┬─────────────┘
           │                      │                    │                    │
           ▼                      ▼                    ▼                    ▼
   [Story Templates]         [Groq LLM]         [Session State]       [Audio Gain /     
                          (Llama 3.3 70B)                              Normalization]
                                                                            │
                                                                            ▼
                                                                     [Google GenAI TTS]
                                                                     (Gemini 2.5 Flash)
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 1. Luồng Hoạt Động (Conversation Flow)
1. **Calibrate & Auto-record**: Client (`test.py`) hiệu chỉnh mic để đo độ ồn nền (noise floor) và tự động tính toán ngưỡng âm thanh (`threshold`). Sau đó, chương trình lắng nghe giọng nói của bé, tự động ghi âm và dừng lại sau khi bé im lặng 5 giây (`SILENCE_TIMEOUT`).
2. **Speech-to-Text (STT)**: Audio thô của bé được chuẩn hóa biên độ (boost gain) để tăng độ rõ, sau đó gửi lên API `/api/transcribe` của server. Server sử dụng **Groq Whisper-large-v3-turbo** để chuyển đổi giọng nói thành văn bản tiếng Việt.
3. **Intent Classification**: Ý định của bé được phân loại bằng **Groq LLM (Llama-3.3-70b-versatile)** thành 4 nhóm chính:
   - `ANSWER`: Bé đang trả lời câu hỏi của cô giáo.
   - `QUESTION`: Bé đang hỏi thêm về nội dung truyện.
   - `CONFIRM`: Bé xác nhận đã hiểu (ví dụ: "dạ", "vâng", "con hiểu rồi").
   - `CONFUSED`: Bé không biết câu trả lời hoặc xin gợi ý ("con quên rồi", "không biết").
4. **Chat & LLM Response**: Dựa trên ý định và lịch sử trò chuyện, hệ thống sử dụng Llama 3.3 để sinh câu trả lời đóng vai cô giáo ấm áp, ngắn gọn (tối đa 3 câu), không sử dụng ký tự đặc biệt hay bullet points.
5. **Text-to-Speech (TTS)**: Phản hồi văn bản của cô giáo được chuyển thành giọng nói qua `/api/speak` sử dụng **Gemini 2.5 Flash's Experimental Audio Modality** với các phong cách giọng nói phù hợp (ví dụ: khích lệ, an ủi, giải thích).
6. **Play Audio**: Client nhận file âm thanh WAV trả về từ server và phát qua loa của thiết bị.

---

## 📂 Danh Mục Các Service Cần Tối Ưu

Hệ thống hiện tại có các file service nằm trong thư mục `services/`:

1. **`content_service.py`**: Quản lý template của cô giáo và làm sạch văn bản thô.
2. **`llm_service.py`**: Xử lý việc gọi API Groq để nhận câu trả lời từ Llama và phân loại ý định của bé.
3. **`session_service.py`**: Quản lý lịch sử hội thoại của session để tránh quá tải token.
4. **`stt_service.py`**: Chuẩn hóa âm thanh đầu vào và chuyển đổi thành văn bản.
5. **`tts_service.py`**: Gọi Gemini GenAI để tổng hợp giọng đọc cô giáo theo các phong cách khác nhau.

---

## 🚀 Kế Hoạch Tối Ưu Hóa Dành Cho Bạn

Chúng ta sẽ tối ưu toàn bộ các file service trên theo các tiêu chuẩn cao cấp nhất:
- **Async API & Connection Pooling**: Chuyển đổi các API mạng (Groq, Google GenAI) sang Async Client (`AsyncGroq`, `client.aio`) để cải thiện hiệu suất xử lý đồng thời, giảm thiểu độ trễ phản hồi của bot.
- **Dynamic Gain Normalization**: Thay thế việc nhân âm lượng cố định (x4) bằng thuật toán chuẩn hóa biên độ động (Adaptive Normalization) giúp âm thanh đầu vào của bé luôn rõ nét, không bị rè hoặc méo tiếng khi bé nói to/nhỏ.
- **Smart TTS Audio Cache**: Lưu trữ cache cục bộ các đoạn âm thanh TTS phổ biến (như lời chào, nhắc nhở, khen ngợi) bằng hash MD5. Điều này giúp giảm 100% thời gian phản hồi (0ms latency) và tiết kiệm chi phí gọi API Google Gemini cho các câu thoại lặp đi lặp lại.
- **Robust Intent Parsing**: Cải tiến prompt phân loại ý định và bộ parse kết quả đầu ra của LLM, giúp hệ thống nhận diện cực kỳ chính xác các câu nói ngây ngô của trẻ em (ví dụ: "con hổng biết", "dạ vâng ạ").
- **Thread-safe Session Storage**: Đảm bảo bộ nhớ lịch sử hoạt động trơn tru trong môi trường bất đồng bộ nhiều người dùng đồng thời.


---

## ⚡ Quy Tắc Làm Việc Với AI Coding Agent (Giảm Token & Trả lời tập trung)

### Mục tiêu
Ưu tiên sửa code và kết quả thực tế. Giảm tối đa giải thích dài, báo cáo dư thừa và xác nhận lặp lại.

### Luật bắt buộc
- Không chào hỏi.
- Không nhắc lại yêu cầu của người dùng.
- Không mô tả điều đã hiểu.
- Không giải thích kế hoạch nếu chưa được hỏi.
- Không viết báo cáo triển khai dài.
- Không sinh tóm tắt lặp lại yêu cầu.
- Ưu tiên: **Code → Diff → Kết quả → Hết**.

### Format phản hồi khi CHƯA sửa code

```txt
Sẽ sửa:
- file_a.py
- file_b.py

Thay đổi:
- Mô tả ngắn (1–2 ý)

Chờ xác nhận.
```

### Format phản hồi khi ĐÃ sửa code

```txt
Đã sửa:
✓ file_a.py
✓ file_b.py

Kết quả:
✓ Tính năng hoạt động
✓ Logic kiểm thử đạt

Test:
PASS
```

### Giới hạn phản hồi
- Tối đa ~80 từ nếu không cần giải thích.
- Chỉ liệt kê:
  1. File thay đổi
  2. Thay đổi chính
  3. Trạng thái test
  4. Vấn đề còn lại (nếu có)

### Cấm
Không sinh các cụm như:
- "Tôi đã hiểu rõ vấn đề..."
- "Dưới đây là kế hoạch..."
- "Tôi đã hoàn tất..."
- "Kết quả triển khai chi tiết..."

Ưu tiên phản hồi ngắn, giống log kỹ thuật hơn báo cáo quản lý dự án.
