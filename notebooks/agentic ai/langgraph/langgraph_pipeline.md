# LangGraph: Input, Pipeline, Output

## 1. Tổng quan

LangGraph là một framework dùng để xây dựng workflow cho ứng dụng AI theo dạng **graph**.  
Thay vì chỉ gọi LLM một lần, LangGraph cho phép ta chia hệ thống thành nhiều bước xử lý khác nhau.

Một workflow trong LangGraph thường có dạng:

```text
Input
  ↓
State
  ↓
Graph Pipeline
  ↓
Nodes
  ↓
Edges
  ↓
Output
```

Trong đó:

- **Input** là dữ liệu ban đầu đưa vào graph.
- **Pipeline** là luồng xử lý gồm nhiều node và edge.
- **Output** là kết quả cuối cùng sau khi graph chạy xong.

---

## 2. Input trong LangGraph

### 2.1 Input là gì?

Input trong LangGraph thường là một dictionary chứa dữ liệu ban đầu.  
Dữ liệu này sẽ được đưa vào **State** để các node trong graph cùng sử dụng.

Ví dụ:

```python
input_data = {
    "question": "Giải thích transaction trong PostgreSQL."
}
```

Khi truyền vào graph:

```python
result = graph.invoke(input_data)
```

LangGraph sẽ lấy input này làm trạng thái ban đầu của workflow.

---

### 2.2 State là gì?

State là nơi lưu trữ toàn bộ dữ liệu trong quá trình graph chạy.

Ví dụ:

```python
from typing_extensions import TypedDict

class State(TypedDict):
    question: str
    answer: str
```

Trong ví dụ này:

- `question`: câu hỏi đầu vào của người dùng.
- `answer`: câu trả lời được tạo ra sau khi xử lý.

Input ban đầu có thể chỉ có `question`, còn `answer` sẽ được thêm vào sau khi node xử lý xong.

---

## 3. Pipeline trong LangGraph

### 3.1 Pipeline là gì?

Pipeline trong LangGraph là luồng xử lý được thiết kế bằng graph.

Một pipeline gồm:

- **Node**: bước xử lý cụ thể.
- **Edge**: đường nối giữa các node.
- **START**: điểm bắt đầu của graph.
- **END**: điểm kết thúc của graph.

Ví dụ pipeline đơn giản:

```text
START
  ↓
answer_question
  ↓
END
```

---

### 3.2 Node là gì?

Node là một hàm xử lý dữ liệu trong State.

Ví dụ:

```python
def answer_question(state: State) -> dict:
    response = model.invoke(state["question"])
    return {
        "answer": response.content
    }
```

Node này nhận vào `state`, đọc `question`, gọi model để tạo câu trả lời, rồi trả về `answer`.

---

### 3.3 Edge là gì?

Edge dùng để nối các node lại với nhau, tạo thành luồng xử lý.

Ví dụ:

```python
builder.add_edge(START, "answer_question")
builder.add_edge("answer_question", END)
```

Ý nghĩa:

```text
START → answer_question → END
```

---

## 4. Output trong LangGraph

### 4.1 Output là gì?

Output là State cuối cùng sau khi graph chạy xong.

Ví dụ:

```python
{
    "question": "Giải thích transaction trong PostgreSQL.",
    "answer": "Transaction là một nhóm thao tác SQL được thực hiện như một đơn vị..."
}
```

LangGraph không chỉ trả về câu trả lời, mà trả về toàn bộ State cuối cùng.

---

## 5. Ví dụ đầy đủ

```python
from typing_extensions import TypedDict
from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END


class State(TypedDict, total=False):
    question: str
    answer: str


model = init_chat_model("openai:gpt-5.4")


def answer_question(state: State) -> dict:
    response = model.invoke(
        [
            {"role": "system", "content": "Bạn là trợ lý học database."},
            {"role": "user", "content": state["question"]},
        ]
    )

    return {
        "answer": response.content
    }


builder = StateGraph(State)

builder.add_node("answer_question", answer_question)

builder.add_edge(START, "answer_question")
builder.add_edge("answer_question", END)

graph = builder.compile()

result = graph.invoke({
    "question": "Giải thích transaction trong PostgreSQL."
})

print(result["answer"])
```

---

## 6. Luồng hoạt động của ví dụ

```text
User nhập câu hỏi
  ↓
Input được đưa vào State
  ↓
Graph bắt đầu từ START
  ↓
Node answer_question xử lý câu hỏi
  ↓
Model tạo câu trả lời
  ↓
Node trả về answer
  ↓
LangGraph cập nhật State
  ↓
Graph đi đến END
  ↓
Trả về Output
```

---

## 7. Minh họa Input, Pipeline, Output

### Input

```python
{
    "question": "Giải thích transaction trong PostgreSQL."
}
```

### Pipeline

```text
START
  ↓
answer_question
  ↓
END
```

### Output

```python
{
    "question": "Giải thích transaction trong PostgreSQL.",
    "answer": "Transaction là một nhóm thao tác SQL..."
}
```

---

## 8. Bảng tóm tắt

| Thành phần | Vai trò | Ví dụ |
|---|---|---|
| Input | Dữ liệu đầu vào | Câu hỏi của user |
| State | Bộ nhớ chung của graph | question, answer, messages |
| Node | Bước xử lý | Gọi LLM, gọi tool, truy xuất database |
| Edge | Điều hướng luồng chạy | START → Node → END |
| Pipeline | Toàn bộ workflow | Chuỗi node được nối với nhau |
| Output | State cuối cùng | Câu trả lời cuối cùng |

---

## 9. Mô hình thực tế hơn

Trong hệ thống lớn, LangGraph thường không chỉ có một node.  
Một pipeline thực tế có thể gồm nhiều bước:

```text
Input
  ↓
Intent Detection
  ↓
Retrieval
  ↓
Tool Calling
  ↓
LLM Reasoning
  ↓
Final Response
  ↓
Output
```

Ý nghĩa từng bước:

| Bước | Chức năng |
|---|---|
| Intent Detection | Xác định người dùng muốn gì |
| Retrieval | Tìm dữ liệu liên quan |
| Tool Calling | Gọi API, database hoặc công cụ bên ngoài |
| LLM Reasoning | Model xử lý và suy luận |
| Final Response | Tạo câu trả lời cuối cùng |

---

## 10. Kết luận

LangGraph có thể hiểu đơn giản như sau:

```text
Input = dữ liệu ban đầu

Pipeline = luồng xử lý bằng graph

Output = State cuối cùng sau khi xử lý
```

Cách tư duy quan trọng khi dùng LangGraph là:

```text
Không chỉ viết một hàm gọi LLM,
mà thiết kế một workflow gồm nhiều bước xử lý rõ ràng.
```

LangGraph phù hợp khi cần xây dựng:

- AI Agent
- Multi-agent system
- Chatbot có memory
- Workflow có nhiều bước
- Hệ thống có tool calling
- Pipeline xử lý dữ liệu tự động
