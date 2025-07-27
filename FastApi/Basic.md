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
    items_db.append(new_item)
    return {"message": "Item created", "item": new_item}

@app.get("/items/{item_id}")
def get_item(item_id: int):
    for item in items_db:
        if item["id"] == item_id:
            return item
    raise HTTPException(status_code=404, detail="Item not found")

@app.delete("/items/{item_id}")
def delete_item(item_id: int):
    for i, item in enumerate(items_db):
        if item["id"] == item_id:
            deleted_item = items_db.pop(i)
            return {"message": "Item deleted", "item": deleted_item}
    raise HTTPException(status_code=404, detail="Item not found")
```

### What You'll Learn:
- Different HTTP methods (GET, POST, DELETE)
- Handling HTTP exceptions
- Working with path parameters
- Basic CRUD operations

---

## Lesson 3: Pydantic Models and Data Validation

### Code Example:
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, validator
from typing import Optional, List
from datetime import datetime

app = FastAPI()

class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    category: str
    
    @validator('price')
    def price_must_be_positive(cls, v):
        if v <= 0:
            raise ValueError('Price must be positive')
        return v

class ItemResponse(BaseModel):
    id: int
    name: str
    description: Optional[str]
    price: float
    category: str
    created_at: datetime

# In-memory database
items_db: List[ItemResponse] = []

@app.post("/items", response_model=ItemResponse)
def create_item(item: Item):
    item_id = len(items_db) + 1
    new_item = ItemResponse(
        id=item_id,
        **item.dict(),
        created_at=datetime.now()
    )
    items_db.append(new_item)
    return new_item

@app.get("/items", response_model=List[ItemResponse])
def get_items():
    return items_db

@app.get("/items/{item_id}", response_model=ItemResponse)
def get_item(item_id: int):
    for item in items_db:
        if item.id == item_id:
            return item
    raise HTTPException(status_code=404, detail="Item not found")
```

### What You'll Learn:
- Creating Pydantic models for data validation
- Using type hints for automatic validation
- Custom validators
- Response models
- Optional fields with default values

---

## Lesson 4: Query Parameters and Request Body

### Code Example:
```python
from fastapi import FastAPI, Query
from pydantic import BaseModel
from typing import Optional, List
from enum import Enum

app = FastAPI()

class CategoryEnum(str, Enum):
    electronics = "electronics"
    clothing = "clothing"
    books = "books"
    food = "food"

class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    category: CategoryEnum

items_db: List[Item] = [
    Item(name="Laptop", price=999.99, category="electronics"),
    Item(name="T-Shirt", price=19.99, category="clothing"),
    Item(name="Python Book", price=29.99, category="books"),
]

@app.get("/items")
def get_items(
    category: Optional[CategoryEnum] = None,
    min_price: Optional[float] = Query(None, ge=0),
    max_price: Optional[float] = Query(None, ge=0),
    limit: int = Query(10, le=100)
):
    filtered_items = items_db
    
    if category:
        filtered_items = [item for item in filtered_items if item.category == category]
    
    if min_price is not None:
        filtered_items = [item for item in filtered_items if item.price >= min_price]
    
    if max_price is not None:
        filtered_items = [item for item in filtered_items if item.price <= max_price]
    
    return filtered_items[:limit]

@app.post("/items")
def create_item(item: Item):
    items_db.append(item)
    return {"message": "Item created successfully", "item": item}
```

### What You'll Learn:
- Query parameters with validation
- Using `Query` for advanced parameter validation
- Enums in Pydantic models
- Filtering data based on query parameters

---

## Lesson 5: Headers, Cookies, and Dependencies

### Code Example:
```python
from fastapi import FastAPI, Header, Cookie, Depends, HTTPException
from typing import Optional

app = FastAPI()

# Simple authentication dependency
def get_current_user(authorization: Optional[str] = Header(None)):
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Authentication required")
    
    token = authorization.split(" ")[1]
    if token != "secret-token":
        raise HTTPException(status_code=401, detail="Invalid token")
    
    return {"username": "john_doe", "email": "john@example.com"}

@app.get("/profile")
def get_profile(current_user: dict = Depends(get_current_user)):
    return {"user": current_user}

@app.get("/items")
def get_items(
    user_agent: Optional[str] = Header(None),
    session_id: Optional[str] = Cookie(None)
):
    return {
        "message": "Items retrieved",
        "user_agent": user_agent,
        "session_id": session_id
    }

@app.post("/set-cookie")
def set_cookie(response):
    response.set_cookie(key="session_id", value="abc123", max_age=3600)
    return {"message": "Cookie set"}
```

### What You'll Learn:
- Working with headers and cookies
- Dependency injection with `Depends`
- Simple authentication patterns
- Extracting request metadata

---

## Lesson 6: File Upload and Form Data

### Code Example:
```python
from fastapi import FastAPI, File, UploadFile, Form
from typing import List
import shutil
import os

app = FastAPI()

# Create uploads directory if it doesn't exist
os.makedirs("uploads", exist_ok=True)

@app.post("/upload-file")
async def upload_file(file: UploadFile = File(...)):
    file_path = f"uploads/{file.filename}"
    
    with open(file_path, "wb") as buffer:
        shutil.copyfileobj(file.file, buffer)
    
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": file.size,
        "message": "File uploaded successfully"
    }

@app.post("/upload-multiple-files")
async def upload_multiple_files(files: List[UploadFile] = File(...)):
    uploaded_files = []
    
    for file in files:
        file_path = f"uploads/{file.filename}"
        with open(file_path, "wb") as buffer:
            shutil.copyfileobj(file.file, buffer)
        
        uploaded_files.append({
            "filename": file.filename,
            "size": file.size
        })
    
    return {"uploaded_files": uploaded_files}

@app.post("/submit-form")
async def submit_form(
    name: str = Form(...),
    email: str = Form(...),
    age: int = Form(...),
    file: UploadFile = File(None)
):
    result = {
        "name": name,
        "email": email,
        "age": age
    }
    
    if file:
        result["uploaded_file"] = file.filename
    
    return result
```

### What You'll Learn:
- Handling file uploads
- Working with form data
- Multiple file uploads
- Combining form fields with file uploads

---

## Lesson 7: Error Handling and Custom Exceptions

### Code Example:
```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel, ValidationError
import logging

app = FastAPI()

# Custom exception classes
class ItemNotFoundError(Exception):
    def __init__(self, item_id: int):
        self.item_id = item_id

class InsufficientStockError(Exception):
    def __init__(self, item_id: int, requested: int, available: int):
        self.item_id = item_id
        self.requested = requested
        self.available = available

# Custom exception handlers
@app.exception_handler(ItemNotFoundError)
async def item_not_found_handler(request: Request, exc: ItemNotFoundError):
    return JSONResponse(
        status_code=404,
        content={
            "error": "Item not found",
            "item_id": exc.item_id,
            "message": f"Item with ID {exc.item_id} does not exist"
        }
    )

@app.exception_handler(InsufficientStockError)
async def insufficient_stock_handler(request: Request, exc: InsufficientStockError):
    return JSONResponse(
        status_code=400,
        content={
            "error": "Insufficient stock",
            "item_id": exc.item_id,
            "requested": exc.requested,
            "available": exc.available
        }
    )

class Item(BaseModel):
    name: str
    price: float
    stock: int

# Sample data
items_db = {
    1: Item(name="Laptop", price=999.99, stock=5),
    2: Item(name="Mouse", price=25.99, stock=0)
}

@app.get("/items/{item_id}")
def get_item(item_id: int):
    if item_id not in items_db:
        raise ItemNotFoundError(item_id)
    return items_db[item_id]

@app.post("/items/{item_id}/purchase")
def purchase_item(item_id: int, quantity: int):
    if item_id not in items_db:
        raise ItemNotFoundError(item_id)
    
    item = items_db[item_id]
    if item.stock < quantity:
        raise InsufficientStockError(item_id, quantity, item.stock)
    
    item.stock -= quantity
    return {
        "message": "Purchase successful",
        "item": item.name,
        "quantity": quantity,
        "remaining_stock": item.stock
    }
```

### What You'll Learn:
- Creating custom exception classes
- Custom exception handlers
- Proper error responses
- HTTP status codes

---

## Lesson 8: Background Tasks and Async Operations

### Code Example:
```python
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel
import asyncio
import smtplib
from email.mime.text import MIMEText
import logging

app = FastAPI()

class EmailRequest(BaseModel):
    to_email: str
    subject: str
    message: str

# Simulate sending email (replace with real implementation)
def send_email_background(email_data: EmailRequest):
    # Simulate time-consuming email sending
    import time
    time.sleep(2)  # Simulate delay
    
    logging.info(f"Email sent to {email_data.to_email}")
    print(f"📧 Email sent to: {email_data.to_email}")
    print(f"📧 Subject: {email_data.subject}")
    print(f"📧 Message: {email_data.message}")

def process_data_background(data: dict):
    # Simulate data processing
    import time
    time.sleep(3)
    
    logging.info(f"Data processed: {data}")
    print(f"🔄 Processed data for user: {data.get('user_id')}")

@app.post("/send-email")
async def send_email(email_data: EmailRequest, background_tasks: BackgroundTasks):
    # Add background task
    background_tasks.add_task(send_email_background, email_data)
    
    return {
        "message": "Email is being sent in the background",
        "email": email_data.to_email
    }

@app.post("/process-data")
async def process_data(data: dict, background_tasks: BackgroundTasks):
    # Immediate response
    response_data = {"status": "accepted", "data_id": data.get("id", "unknown")}
    
    # Process data in background
    background_tasks.add_task(process_data_background, data)
    
    return response_data

# Async endpoint example
@app.get("/async-operation")
async def async_operation():
    # Simulate async database call
    await asyncio.sleep(1)
    
    return {"message": "Async operation completed", "data": "some_result"}
```

### What You'll Learn:
- Background tasks for time-consuming operations
- Async/await patterns
- Non-blocking operations
- Immediate responses with background processing

---

## Practice Exercises

### Exercise 1: Build a Simple Todo API
Create a Todo API with the following endpoints:
- `POST /todos` - Create a new todo
- `GET /todos` - Get all todos (with optional filtering)
- `GET /todos/{todo_id}` - Get a specific todo
- `PUT /todos/{todo_id}` - Update a todo
- `DELETE /todos/{todo_id}` - Delete a todo

### Exercise 2: User Authentication System
Build a simple user system with:
- User registration
- User login (return a token)
- Protected endpoints that require authentication
- User profile management

### Exercise 3: File Processing API
Create an API that:
- Accepts CSV file uploads
- Processes the data in the background
- Provides endpoints to check processing status
- Returns processed results

## Next Steps

1. **Database Integration**: Learn to integrate with databases using SQLAlchemy or databases library
2. **Testing**: Write tests using pytest and FastAPI's test client
3. **Deployment**: Deploy your FastAPI apps using Docker, Heroku, or cloud platforms
4. **Advanced Features**: Explore middleware, WebSockets, GraphQL integration
5. **Security**: Implement OAuth2, JWT tokens, and other security features

## Useful Resources

- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Pydantic Documentation](https://pydantic-docs.helpmanual.io/)
- [Uvicorn Documentation](https://www.uvicorn.org/)

Happy coding! 🚀
