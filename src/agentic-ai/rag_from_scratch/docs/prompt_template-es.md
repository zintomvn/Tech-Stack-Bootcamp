In LangChain, a **Prompt Template** is a reusable structure for dynamically creating prompts. It usually includes the following components.

## 1. Template — the prompt content

The template is the main text that tells the LLM what to do.

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

In this example:

- `Answer the user's question...` is the instruction.
- `{context}` is the retrieved information.
- `{question}` is the user's input.
- `Answer:` indicates where the model should begin its response.

LangChain replaces variables inside `{}` with actual values when the chain runs.

## 2. Input variables

Placeholders such as the following are called **input variables**:

```text
{context}
{question}
{language}
{format}
```

Example:

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

You can invoke the prompt like this:

```python
result = prompt.invoke({
    "context": "Task decomposition divides a complex task into smaller tasks.",
    "question": "What is task decomposition?",
    "language": "English"
})
```

The final prompt will look similar to this:

```text
Use the following context to answer the question.

Context:
Task decomposition divides a complex task into smaller tasks.

Question:
What is task decomposition?

Answer in English.
```

## 3. Instructions

Instructions define what the model should do and how it should behave.

Example:

```text
Use only the provided context.
Do not invent information.
If the answer is not in the context, say that you do not know.
Answer concisely.
```

In a RAG system, clear instructions are important because they help reduce hallucinations.

A stronger prompt may look like this:

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

## 4. Context

The `context` variable usually contains additional information from sources such as:

- A vector database
- A retriever
- A PDF file
- A relational database
- An API
- Conversation history
- A search engine

In the following RAG chain:

```python
{"context": retriever | format_docs, "question": RunnablePassthrough()}
```

this part:

```python
retriever | format_docs
```

produces the value for:

```text
{context}
```

Meanwhile:

```python
RunnablePassthrough()
```

passes the original user query into:

```text
{question}
```

## 5. Output instructions

A prompt can also specify how the LLM should format its answer.

For plain text:

```text
Answer in no more than three sentences.
```

For JSON:

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

When using an `f-string` style template, braces are used for variables. Therefore, literal JSON braces often need to be written as `{{` and `}}`.

# Two common Prompt Template types

## `PromptTemplate`

`PromptTemplate` is used for a prompt represented as a single text string.

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

A shorter example:

```python
prompt = PromptTemplate.from_template(
    "Explain {topic} to a {level} student."
)
```

Invocation:

```python
prompt.invoke({
    "topic": "RAG",
    "level": "beginner"
})
```

## `ChatPromptTemplate`

`ChatPromptTemplate` is designed for chat models and divides a prompt into messages with different roles.

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

LangChain creates messages similar to the following:

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

# Roles in `ChatPromptTemplate`

## System message

The system message defines the model's role, rules, and general behavior.

```python
("system", """
You are an AI teaching assistant.
Explain concepts clearly and provide examples.
Do not invent information.
""")
```

A system message usually contains:

- The model's role
- Rules
- Response style
- Constraints
- Output format

## Human message

The human message contains the user's question or instruction.

```python
("human", """
Context:
{context}

Question:
{question}
""")
```

## AI message

An AI message can be used to provide an example answer.

```python
("ai", "Task decomposition means dividing a large task into smaller subtasks.")
```

This is commonly used in few-shot prompting.

## `MessagesPlaceholder`

`MessagesPlaceholder` is used to insert conversation history into the prompt.

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

Invocation:

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

LangChain inserts all messages from `chat_history` at the position of `MessagesPlaceholder`.

# A complete RAG prompt structure

A RAG prompt usually includes six main parts:

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

The components are:

| Component     | Example                                  | Purpose                        |
| ------------- | ---------------------------------------- | ------------------------------ |
| Role          | `You are a question-answering assistant` | Defines the model's role       |
| Instruction   | `Use only the provided context`          | Tells the model what to do     |
| Constraints   | `Do not invent information`              | Limits the model's behavior    |
| Context       | `{context}`                              | Contains retrieved documents   |
| User input    | `{question}`                             | Contains the user's query      |
| Output format | `Answer`, `Evidence`                     | Defines the response structure |

# The prompt in a RAG chain

Consider the following chain:

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

Assume the prompt is:

```python
prompt = ChatPromptTemplate.from_template("""
Answer the question based only on the following context:

{context}

Question:
{question}
""")
```

When the chain is invoked:

```python
response = rag_chain.invoke(
    "What is Task Decomposition?"
)
```

LangChain processes it like this:

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

The key names in the chain:

```python
"context"
"question"
```

must match the placeholders in the prompt:

```text
{context}
{question}
```

For example, if the prompt expects:

```text
{user_query}
```

but the chain provides:

```python
"question"
```

LangChain will raise an error because the `user_query` variable is missing.

In summary, a good Prompt Template usually contains:

```text
Role
+ Instructions
+ Constraints
+ Context
+ User input
+ Examples, when needed
+ Output format
```

At the LangChain object level, it usually contains:

```text
Template or messages
+ Input variables
+ Partial variables, when needed
+ Message placeholders, when needed
+ Template format
```
