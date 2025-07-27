# FastAPI Learning Guide with Examples

## Prerequisites
- Basic Python knowledge
- Understanding of HTTP methods (GET, POST, PUT, DELETE)
- Basic understanding of APIs

## Installation
```bash
pip install fastapi uvicorn
```

---

## Lesson 1: Hello World - Your First FastAPI App

### Code Example:
```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello, World!"}

@app.get("/hello/{name}")
def say_hello(name: str):
    return {"message": f"Hello, {name}!"}
```

### How to Run:
```bash
uvicorn main:app --reload
```

### What You'll Learn:
- Creating a FastAPI instance
- Basic route decoration with `@app.get()`
- Path parameters
- Returning JSON responses

### Try It:
- Visit `http://127.0.0.1:8000` 
- Visit `http://127.0.0.1:8000/hello/Alice`
- Check automatic docs at `http://127.0.0.1:8000/docs`

---

## Lesson 2: HTTP Methods and Status Codes

### Code Example:
```python
from fastapi import FastAPI, HTTPException
from typing import Dict, List

app = FastAPI()

# In-memory database
items_db: List[Dict] = []

@app.get("/items")
def get_items():
    return {"items": items_db}

@app.post("/items")
def create_item(item: Dict):
    item_id = len(items_db) + 1
    new_item = {"id": item_id, **item}
    items_db.append(n
