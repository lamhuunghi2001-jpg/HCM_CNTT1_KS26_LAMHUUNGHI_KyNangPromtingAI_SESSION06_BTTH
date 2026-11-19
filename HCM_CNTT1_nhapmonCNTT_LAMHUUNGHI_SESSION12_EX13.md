# [BTTH] Bài tập thực hành tổng hợp

## 1. Bài toán

Sinh viên viết bài giải thích một khái niệm (ví dụ vòng lặp for). Hệ thống cần làm hai việc liên tiếp: giải thích lại khái niệm theo kiểu Feynman để sinh viên thấy chỗ mình còn hổng, rồi chấm bài theo rubric cố định và trả điểm dạng JSON để máy đọc được.

Một prompt duy nhất làm cả hai việc thì kết quả hay lẫn lộn, điểm lúc có lúc không. Nên em tách thành chuỗi 4 bước, bước sau dùng đầu ra của bước trước.

## 2. Thiết kế chuỗi

Bước 0 (code Python, không gọi AI): làm sạch đầu vào. Kiểm tra kiểu dữ liệu, bỏ khoảng trắng thừa, chặn bài rỗng hoặc dưới 20 ký tự, cắt bài ở 1500 ký tự để đỡ tốn token.

Bước 1 (AI): trích khái niệm và ý chính từ bài, trả JSON ngắn.

Bước 2 (AI): gia sư Feynman. Nhận JSON của bước 1 (không nhận lại cả bài gốc nên tiết kiệm token), giải thích cho học sinh lớp 6 bằng một ví dụ đời thường, nêu tối đa 2 chỗ còn hổng.

Bước 3 (AI): chấm rubric, trả JSON. Code kiểm tra đủ tiêu chí và điểm nằm trong khoảng cho phép.

Rubric (tổng 10 điểm): đúng khái niệm 0 đến 4, ví dụ đời thường 0 đến 3, ngôn ngữ đơn giản 0 đến 3.

## 3. Cơ chế bắt lỗi

- Đầu vào sai kiểu, rỗng, quá ngắn: trả về thông báo lỗi, không gọi AI, không crash.
- AI trả JSON kèm chữ thừa: dùng regex lấy khối JSON đầu tiên.
- JSON hỏng hoặc điểm ngoài khoảng (ví dụ cho 9/3): thử lại 1 lần, kèm lý do lỗi trong prompt. Vẫn hỏng thì báo lỗi rõ ràng.
- Toàn bộ chuỗi bọc trong try/except, kết quả luôn có trường ok là true hoặc false.

## 4. Prompt hệ thống

Prompt bước 1:

```
Trích khái niệm và ý chính từ bài làm. Chỉ trả JSON: {"khai_niem": str, "y_chinh": [str]}.
```

Prompt bước 2:

```
Bạn là gia sư Feynman. Giải thích khái niệm cho học sinh lớp 6, dùng 1 ví dụ đời thường, rồi nêu tối đa 2 chỗ bài làm còn hổng.
```

Prompt bước 3:

```
Chấm bài theo rubric: dung_khai_niem 0-4, vi_du_doi_thuong 0-3, ngon_ngu_don_gian 0-3. Chỉ trả JSON: {"diem": {tieu_chi: so}, "nhan_xet": str}.
```

## 5. Code

```python
import json, re

MAX_CHARS = 1500
RUBRIC = {"dung_khai_niem": 4, "vi_du_doi_thuong": 3, "ngon_ngu_don_gian": 3}

P1 = "Trích khái niệm và ý chính từ bài làm. Chỉ trả JSON: {\"khai_niem\": str, \"y_chinh\": [str]}."
P2 = "Bạn là gia sư Feynman. Giải thích khái niệm cho học sinh lớp 6, dùng 1 ví dụ đời thường, rồi nêu tối đa 2 chỗ bài làm còn hổng."
P3 = ("Chấm bài theo rubric: dung_khai_niem 0-4, vi_du_doi_thuong 0-3, ngon_ngu_don_gian 0-3. "
      "Chỉ trả JSON: {\"diem\": {tieu_chi: so}, \"nhan_xet\": str}.")

_calls = {"n": 0}
def call_llm(system, user):
    """Hàm giả lập. Khi có API key thì thay thân hàm này bằng lời gọi thật."""
    if system is P1:
        return '{"khai_niem": "vong lap for", "y_chinh": ["lap qua tung phan tu", "co the dung range"]}'
    if system is P2:
        return "Vòng lặp for giống như cô giáo gọi tên từng bạn trong lớp, lần lượt từng người. Chỗ hổng: chưa nói khi nào vòng lặp dừng."
    _calls["n"] += 1
    if _calls["n"] == 1:  # lần đầu cố tình trả JSON hỏng để thử cơ chế retry
        return 'Đây là kết quả: {"diem": {"dung_khai_niem": 3, "vi_du_doi_thuong": 9}, "nhan_xet": "ok"}'
    return '{"diem": {"dung_khai_niem": 3, "vi_du_doi_thuong": 2, "ngon_ngu_don_gian": 3}, "nhan_xet": "Hiểu đúng ý chính, ví dụ còn ngắn."}'

def clean_input(raw):
    if not isinstance(raw, str):
        raise ValueError("Dữ liệu không phải chuỗi")
    text = re.sub(r"\s+", " ", raw).strip()
    if len(text) < 20:
        raise ValueError("Bài làm quá ngắn hoặc rỗng")
    return text[:MAX_CHARS]

def parse_json(reply):
    m = re.search(r"\{.*\}", reply, re.S)
    if not m:
        raise ValueError("Không tìm thấy JSON")
    return json.loads(m.group(0))

def check_rubric(data):
    diem = data.get("diem")
    if not isinstance(diem, dict) or set(diem) != set(RUBRIC):
        raise ValueError("Thiếu hoặc thừa tiêu chí")
    for k, mx in RUBRIC.items():
        if not isinstance(diem[k], int) or not 0 <= diem[k] <= mx:
            raise ValueError(f"Điểm {k} ngoài khoảng 0-{mx}")
    return data

def check_extract(data):
    if "khai_niem" not in data:
        raise ValueError("Thiếu khai_niem")
    return data

def ask(system, user, check=None, retries=1):
    err = None
    for _ in range(retries + 1):
        try:
            out = call_llm(system, user)
            data = parse_json(out) if check else out
            return check(data) if check else data
        except (ValueError, json.JSONDecodeError) as e:
            err = e
            user += f"\n(Lần trước lỗi: {e}. Trả đúng định dạng.)"
    raise RuntimeError(f"Bước thất bại: {err}")

def run_chain(raw):
    try:
        text = clean_input(raw)
        b1 = ask(P1, text, check=check_extract)
        b2 = ask(P2, json.dumps(b1, ensure_ascii=False))
        b3 = ask(P3, text, check=check_rubric)
        return {"ok": True, "giai_thich": b2, "tong_diem": sum(b3["diem"].values()), **b3}
    except (ValueError, RuntimeError) as e:
        return {"ok": False, "loi": str(e)}

if __name__ == "__main__":
    tests = {
        "bai tot": "Vòng lặp for dùng để lặp qua từng phần tử trong danh sách, ví dụ in tên từng bạn trong lớp.",
        "rong": "   ",
        "sai kieu": None,
        "qua ngan": "for",
    }
    for name, raw in tests.items():
        print(name, "->", json.dumps(run_chain(raw), ensure_ascii=False))
```

## 6. Kết quả chạy thử

Em chạy 4 ca: một bài tốt, một bài rỗng, một đầu vào sai kiểu (None), một bài chỉ có chữ "for". Hàm gọi AI trong file đang là bản giả lập để chạy offline. Ở lần chấm đầu tiên, bản giả lập cố tình trả điểm 9 cho tiêu chí tối đa 3 và thiếu một tiêu chí, để thử retry.

```
bai tot -> {"ok": true, "giai_thich": "Vòng lặp for giống như cô giáo gọi tên từng bạn trong lớp, lần lượt từng người. Chỗ hổng: chưa nói khi nào vòng lặp dừng.", "tong_diem": 8, "diem": {"dung_khai_niem": 3, "vi_du_doi_thuong": 2, "ngon_ngu_don_gian": 3}, "nhan_xet": "Hiểu đúng ý chính, ví dụ còn ngắn."}
rong -> {"ok": false, "loi": "Bài làm quá ngắn hoặc rỗng"}
sai kieu -> {"ok": false, "loi": "Dữ liệu không phải chuỗi"}
qua ngan -> {"ok": false, "loi": "Bài làm quá ngắn hoặc rỗng"}
```

Kết quả: bài tốt được 8/10 sau khi tự thử lại 1 lần. Ba ca dị biệt đều trả lỗi gọn, chương trình không bị dừng.

## 7. Nhận xét

Điểm em thấy hiệu quả nhất là bước 0, vì chặn được rác trước khi tốn token. Hạn chế hiện tại: mới thử với bản giả lập, chưa so điểm của AI thật với điểm giáo viên chấm. Nếu có thời gian em sẽ thử 10 bài thật rồi đối chiếu.
