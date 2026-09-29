Trong LangChain, **Prompt Template** là khuôn mẫu dùng để tạo prompt động. Nó thường bao gồm các thành phần chính sau:

## 1. Template — nội dung hướng dẫn

Đây là phần văn bản mô tả cho LLM biết phải làm gì.

```python
template = """
Answer the user's question using the provided context.

Context:
{context}

Question:
{question}

Answer:
"""
```

Trong đó:

- `Answer the user's question...`: instruction
- `{context}`: dữ liệu lấy từ retriever
- `{question}`: câu hỏi người dùng
- `Answer:`: vị trí gợi ý LLM bắt đầu trả lời

LangChain thay các biến nằm trong `{}` bằng dữ liệu thực tế lúc chạy.

## 2. Input variables — các biến đầu vào

Các placeholder như:

```text
{context}
{question}
{language}
{format}
```

được gọi là **input variables**.

Ví dụ:

```python
from langchain_core.prompts import PromptTemplate

prompt = PromptTemplate(
    template="""
Use the following context to answer the question.

Context:
{context}

Question:
{question}

Answer in {language}.
""",
    input_variables=["context", "question", "language"]
)
```

Khi chạy:

```python
result = prompt.invoke({
    "context": "Task decomposition divides a complex task into smaller tasks.",
    "question": "What is task decomposition?",
    "language": "English"
})
```

Prompt hoàn chỉnh sẽ gần giống:

```text
Use the following context to answer the question.

Context:
Task decomposition divides a complex task into smaller tasks.

Question:
What is task decomposition?

Answer in English.
```

## 3. Instructions — yêu cầu LLM thực hiện

Instruction xác định hành vi của model.

Ví dụ:

```text
Use only the provided context.
Do not invent information.
If the answer is not in the context, say that you do not know.
Answer concisely.
```

Trong hệ thống RAG, instruction rất quan trọng để giảm hallucination.

Ví dụ prompt tốt hơn:

```python
template = """
You are a question-answering assistant.

Use only the following context to answer the question.
If the context does not contain enough information, say:
"I don't have enough information to answer."

Context:
{context}

Question:
{question}

Answer:
"""
```

## 4. Context — dữ liệu bổ sung

`context` thường là nội dung lấy từ:

- Vector database
- Retriever
- File PDF
- Database
- API
- Conversation history
- Search engine

Trong RAG chain:

```python
{"context": retriever | format_docs, "question": RunnablePassthrough()}
```

thì:

```python
retriever | format_docs
```

tạo dữ liệu cho biến:

```text
{context}
```

Còn:

```python
RunnablePassthrough()
```

đưa câu hỏi ban đầu vào biến:

```text
{question}
```

## 5. Output instructions — định dạng đầu ra

Prompt có thể yêu cầu LLM trả về theo định dạng cụ thể.

Ví dụ trả về văn bản:

```text
Answer in no more than three sentences.
```

Trả về JSON:

```python
template = """
Analyze the following text:

{text}

Return the result as JSON with this structure:

{{
    "summary": "...",
    "main_topic": "...",
    "confidence": 0.0
}}
"""
```

Lưu ý: khi dùng `f-string` template, dấu `{}` dùng cho biến. Vì vậy JSON literal thường phải viết bằng `{{` và `}}`.

# Hai loại Prompt Template phổ biến

## `PromptTemplate`

Dùng cho prompt dạng một chuỗi văn bản đơn giản.

```python
from langchain_core.prompts import PromptTemplate

prompt = PromptTemplate.from_template(
    """
Context:
{context}

Question:
{question}

Answer:
"""
)
```

Hoặc viết ngắn hơn:

```python
prompt = PromptTemplate.from_template(
    "Explain {topic} to a {level} student."
)
```

Chạy:

```python
prompt.invoke({
    "topic": "RAG",
    "level": "beginner"
})
```

## `ChatPromptTemplate`

Dùng cho chat model và chia prompt thành các message theo role.

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    (
        "system",
        "You are a helpful AI assistant. "
        "Use only the provided context."
    ),
    (
        "human",
        """
Context:
{context}

Question:
{question}
"""
    )
])
```

LangChain sẽ tạo ra:

```text
SystemMessage:
You are a helpful AI assistant.
Use only the provided context.

HumanMessage:
Context:
...

Question:
...
```

# Các role trong `ChatPromptTemplate`

## System message

Đặt vai trò, quy tắc và hành vi tổng quát.

```python
("system", """
You are an AI teaching assistant.
Explain concepts clearly and provide examples.
Do not invent information.
""")
```

System message thường chứa:

- Role
- Quy tắc
- Phong cách trả lời
- Giới hạn
- Output format

## Human message

Chứa câu hỏi hoặc yêu cầu của người dùng.

```python
("human", """
Context:
{context}

Question:
{question}
""")
```

## AI message

Có thể dùng để cung cấp ví dụ câu trả lời mẫu.

```python
("ai", "Task decomposition means dividing a large task into smaller subtasks.")
```

Cách này có thể được dùng trong few-shot prompting.

## MessagesPlaceholder

Dùng để chèn lịch sử hội thoại.

```python
from langchain_core.prompts import (
    ChatPromptTemplate,
    MessagesPlaceholder
)

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    MessagesPlaceholder(variable_name="chat_history"),
    ("human", "{question}")
])
```

Khi invoke:

```python
from langchain_core.messages import HumanMessage, AIMessage

prompt.invoke({
    "chat_history": [
        HumanMessage(content="What is RAG?"),
        AIMessage(content="RAG combines retrieval with generation.")
    ],
    "question": "Why is retrieval necessary?"
})
```

LangChain sẽ chèn toàn bộ lịch sử vào vị trí `MessagesPlaceholder`.

# Cấu trúc prompt RAG hoàn chỉnh

Một prompt RAG thường có sáu phần:

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    (
        "system",
        """
You are a question-answering assistant.

Rules:
1. Use only the provided context.
2. Do not invent information.
3. If the context does not contain the answer, say you do not know.
4. Answer clearly and concisely.
"""
    ),
    (
        "human",
        """
Context:
{context}

Question:
{question}

Return the answer in this format:

Answer:
Evidence:
"""
    )
])
```

Phân tích:

| Thành phần    | Ví dụ                                    | Chức năng               |
| ------------- | ---------------------------------------- | ----------------------- |
| Role          | `You are a question-answering assistant` | Xác định vai trò        |
| Instruction   | `Use only the provided context`          | Yêu cầu model           |
| Constraints   | `Do not invent information`              | Giới hạn hành vi        |
| Context       | `{context}`                              | Tài liệu được truy xuất |
| User input    | `{question}`                             | Câu hỏi người dùng      |
| Output format | `Answer`, `Evidence`                     | Cấu trúc kết quả        |

# Prompt trong chain

```python
rag_chain = (
    {
        "context": retriever | format_docs,
        "question": RunnablePassthrough()
    }
    | prompt
    | llm
    | StrOutputParser()
)
```

Giả sử `prompt` là:

```python
prompt = ChatPromptTemplate.from_template("""
Answer the question based only on the following context:

{context}

Question:
{question}
""")
```

Khi gọi:

```python
response = rag_chain.invoke(
    "What is Task Decomposition?"
)
```

LangChain thực hiện:

```text
Input question
      ↓
"What is Task Decomposition?"
      ↓
┌──────────────────────────────────┐
│ context                          │
│ = retriever(question)            │
│ = format_docs(documents)         │
│                                  │
│ question                         │
│ = original input                 │
└──────────────────────────────────┘
      ↓
prompt.invoke({
    "context": formatted_documents,
    "question": "What is Task Decomposition?"
})
      ↓
LLM
      ↓
StrOutputParser
      ↓
String response
```

Điểm quan trọng là tên key trong chain:

```python
"context"
"question"
```

phải khớp với placeholder trong prompt:

```text
{context}
{question}
```

Nếu prompt dùng:

```text
{user_query}
```

nhưng chain truyền:

```python
"question"
```

thì LangChain sẽ báo lỗi thiếu biến `user_query`.

Tóm lại, một Prompt Template tốt thường gồm:

```text
Role
+ Instructions
+ Constraints
+ Context
+ User input
+ Examples nếu cần
+ Output format
```

Còn ở cấp độ LangChain object, nó thường gồm:

```text
Template/messages
+ Input variables
+ Partial variables nếu có
+ Message placeholders nếu có
+ Template format
```
