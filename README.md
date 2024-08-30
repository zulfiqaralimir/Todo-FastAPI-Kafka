# Todo-FastAPI-Kafka
To build a Todo application using FastAPI with Kafka for message queuing and user authentication, you can follow these steps. This will involve setting up user authentication with JWT tokens, managing Kafka for message passing, and creating endpoints for managing Todos.


To build a Todo application using FastAPI with Kafka for message queuing and user authentication, you can follow these steps. This will involve setting up user authentication with JWT tokens, managing Kafka for message passing, and creating endpoints for managing Todos.

### 1. **Setup the FastAPI Project**

Start by setting up a new FastAPI project:

```bash
mkdir fastapi-todo-kafka
cd fastapi-todo-kafka
python3 -m venv venv
source venv/bin/activate
pip install fastapi uvicorn kafka-python pydantic bcrypt python-jose
```

### 2. **Create the User Model and Authentication**

Create a `models.py` file to define your user model using Pydantic:

```python
from pydantic import BaseModel, EmailStr
from typing import Optional

class User(BaseModel):
    username: str
    email: EmailStr
    full_name: Optional[str] = None
    hashed_password: str
    disabled: Optional[bool] = None
```

Now, create a `auth.py` file to handle JWT authentication:

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext
from datetime import datetime, timedelta
from typing import Optional

SECRET_KEY = "your_secret_key"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password):
    return pwd_context.hash(password)

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt
```

### 3. **Implement Kafka Producer and Consumer**

Create a `kafka_utils.py` file to handle Kafka operations:

```python
from kafka import KafkaProducer, KafkaConsumer
import json

KAFKA_BROKER_URL = 'localhost:9092'
TOPIC_NAME = 'todo_topic'

def get_kafka_producer():
    producer = KafkaProducer(
        bootstrap_servers=KAFKA_BROKER_URL,
        value_serializer=lambda v: json.dumps(v).encode('utf-8')
    )
    return producer

def get_kafka_consumer():
    consumer = KafkaConsumer(
        TOPIC_NAME,
        bootstrap_servers=KAFKA_BROKER_URL,
        auto_offset_reset='earliest',
        enable_auto_commit=True,
        group_id='todo-group',
        value_deserializer=lambda x: json.loads(x.decode('utf-8'))
    )
    return consumer
```

### 4. **Create Todo Model and CRUD Operations**

Create a `todo.py` file to define your Todo model and CRUD operations:

```python
from pydantic import BaseModel
from typing import Optional

class Todo(BaseModel):
    id: int
    title: str
    description: Optional[str] = None
    completed: bool = False
```

### 5. **Setup FastAPI Endpoints**

Create a `main.py` file where you'll define your FastAPI app, endpoints, and integrate the Kafka producer:

```python
from fastapi import FastAPI, Depends, HTTPException, status
from kafka_utils import get_kafka_producer, get_kafka_consumer
from auth import oauth2_scheme, create_access_token
from todo import Todo

app = FastAPI()

todos = []

producer = get_kafka_producer()

@app.post("/token")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user_dict = authenticate_user(form_data.username, form_data.password)
    if not user_dict:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    access_token = create_access_token(data={"sub": user_dict["username"]})
    return {"access_token": access_token, "token_type": "bearer"}

@app.post("/todos/")
async def create_todo(todo: Todo, token: str = Depends(oauth2_scheme)):
    todos.append(todo)
    producer.send(TOPIC_NAME, todo.dict())
    return {"status": "Todo created", "todo": todo}

@app.get("/todos/")
async def read_todos(token: str = Depends(oauth2_scheme)):
    return todos

@app.get("/kafka/todos")
async def get_todos_from_kafka():
    consumer = get_kafka_consumer()
    todos_from_kafka = []
    for message in consumer:
        todos_from_kafka.append(message.value)
    return todos_from_kafka
```

### 6. **Run the Application**

Finally, run your application with Uvicorn:

```bash
uvicorn main:app --reload
```

### 7. **Kafka Setup**

Ensure Kafka and Zookeeper are up and running:

```bash
# Start Zookeeper
zookeeper-server-start.sh config/zookeeper.properties

# Start Kafka
kafka-server-start.sh config/server.properties
```

### 8. **Testing the Endpoints**

- **Token Generation**: Use the `/token` endpoint to get a JWT token.
- **Todo Creation**: Use the `/todos/` endpoint to create a new todo item.
- **Read Todos**: Use the `/todos/` endpoint to fetch all todos.
- **Kafka Todos**: Use the `/kafka/todos` endpoint to fetch todos directly from Kafka.

This setup provides a basic framework for a Todo application with user authentication and Kafka for message queuing. You can further extend it by adding more features like Todo deletion, updating, and more Kafka consumers for processing the Todo messages.
