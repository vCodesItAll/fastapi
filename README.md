## Setup
# FastAPI To Do API

- GitHub: https://github.com/travisluong/fullstackbook-todo-fastapi
- YouTube: [Full Stack FastAPI + NextJS JWT Authentication Tutorial](https://youtu.be/InzrcSk_9YU)
- YouTube: [Full Stack FastAPI + VanillaJS Tutorial](https://youtu.be/p9YTVAq472E)
- YouTube: [Full Stack FastAPI + NuxtJS Tutorial](https://youtu.be/J9EbxoJGwbI)
- YouTube: [Full Stack FastAPI + Next.js Tutorial](https://youtu.be/HJ314MNcCck)

## Command Line

```bash title="Terminal"
pip install "uvicorn[standard]" "python-jose[cryptography]" "passlib[bcrypt]" python-multipart
pip freeze > requirements.txt
uvicorn main:app --reload
```

## Configuration

```txt title=".env.example"
DATABASE_HOST=localhost
DATABASE_NAME=postgres
DATABASE_USER=postgres
DATABASE_PASSWORD=
DATABASE_PORT=5432
APP_NAME="FastAPI Auth Baseline"
```

```python title="config.py"
from pydantic import BaseSettings


class Settings(BaseSettings):
    app_name: str = "Awesome API"

    class Config:
        env_file = ".env"
```

## Entry Point / CORS / Exception Handler

```python title="main.py"
from functools import lru_cache
from typing import Union

from fastapi import FastAPI, Depends
from fastapi.responses import PlainTextResponse
from starlette.exceptions import HTTPException as StarletteHTTPException
from fastapi.middleware.cors import CORSMiddleware

from routers import todos

import config

app = FastAPI()
app.include_router(todos.router)


origins = [
    "http://localhost:3000",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


@app.exception_handler(StarletteHTTPException)
async def http_exception_handler(request, exc):
    print(f"{repr(exc)}")
    return PlainTextResponse(str(exc.detail), status_code=exc.status_code)

@lru_cache()
def get_settings():
    return config.Settings()


@app.get("/")
def read_root(settings: config.Settings = Depends(get_settings)):
    print(settings.app_name)
    return "Hello World"


@app.get("/items/{item_id}")
def read_item(item_id: int, q: Union[str, None] = None):
    return {"item_id": item_id, "q": q}
```

## Database Migrations

```python title="alembic/env.py"
import os
from logging.config import fileConfig

from sqlalchemy import engine_from_config
from sqlalchemy import pool

from alembic import context
from dotenv import load_dotenv

load_dotenv()

# this is the Alembic Config object, which provides
# access to the values within the .ini file in use.
config = context.config

config.set_main_option('sqlalchemy.url', f"postgresql://{os.environ['DATABASE_USER']}:@{os.environ['DATABASE_HOST']}/{os.environ['DATABASE_NAME']}")

# Interpret the config file for Python logging.
# This line sets up loggers basically.
fileConfig(config.config_file_name)

# add your model's MetaData object here
# for 'autogenerate' support
# from myapp import mymodel
# target_metadata = mymodel.Base.metadata
target_metadata = None

# other values from the config, defined by the needs of env.py,
# can be acquired:
# my_important_option = config.get_main_option("my_important_option")
# ... etc.


def run_migrations_offline():
    """Run migrations in 'offline' mode.

    This configures the context with just a URL
    and not an Engine, though an Engine is acceptable
    here as well.  By skipping the Engine creation
    we don't even need a DBAPI to be available.

    Calls to context.execute() here emit the given string to the
    script output.

    """
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online():
    """Run migrations in 'online' mode.

    In this scenario we need to create an Engine
    and associate a connection with the context.

    """
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection, target_metadata=target_metadata
        )

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

```python title="alembic/versions/049c5ab5051e_create_todos_table.py"
"""create todos table

Revision ID: 049c5ab5051e
Revises: 
Create Date: 2022-07-27 16:52:56.499567

"""
from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision = '049c5ab5051e'
down_revision = None
branch_labels = None
depends_on = None


def upgrade():
    op.execute("""
        create table todos (
            id bigserial primary key,
            name text,
            completed boolean not null default false
        );
    """)


def downgrade():
    op.execute("drop table todos;")
```

## Schemas

```python title="schemas.py"
from pydantic import BaseModel

class ToDoRequest(BaseModel):
    name: str
    completed: bool

class ToDoResponse(BaseModel):
    name: str
    completed: bool
    id: int

    class Config:
        orm_mode = True
```

## ORM

```python title="database.py"
import os
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from dotenv import load_dotenv

load_dotenv()

SQLALCHEMY_DATABASE_URL = f"postgresql://{os.environ['DATABASE_USER']}:@{os.environ['DATABASE_HOST']}/{os.environ['DATABASE_NAME']}"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL
)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()
```

```python title="models.py"
from sqlalchemy import Boolean, Column, ForeignKey, Integer, String
from sqlalchemy.orm import relationship

from database import Base


class ToDo(Base):
    __tablename__ = "todos"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String)
    completed = Column(Boolean, default=False)
```

```python title="crud.py"
from sqlalchemy.orm import Session
import models, schemas

def create_todo(db: Session, todo: schemas.ToDoRequest):
    db_todo = models.ToDo(name=todo.name, completed=todo.completed)
    db.add(db_todo)
    db.commit()
    db.refresh(db_todo)
    return db_todo

def read_todos(db: Session, completed: bool):
    if completed is None:
        return db.query(models.ToDo).all()
    else:
        return db.query(models.ToDo).filter(models.ToDo.completed == completed).all()

def read_todo(db: Session, id: int):
    return db.query(models.ToDo).filter(models.ToDo.id == id).first()

def update_todo(db: Session, id: int, todo: schemas.ToDoRequest):
    db_todo = db.query(models.ToDo).filter(models.ToDo.id == id).first()
    if db_todo is None:
        return None
    db.query(models.ToDo).filter(models.ToDo.id == id).update({'name': todo.name, 'completed': todo.completed})
    db.commit()
    db.refresh(db_todo)
    return db_todo

def delete_todo(db: Session, id: int):
    db_todo = db.query(models.ToDo).filter(models.ToDo.id == id).first()
    if db_todo is None:
        return None
    db.query(models.ToDo).filter(models.ToDo.id == id).delete()
    db.commit()
    return True
```

## Routers

```python title="routers/todos.py"
from typing import List
from sqlalchemy.orm import Session
from fastapi import APIRouter, Depends, HTTPException, status
import schemas
import crud
from database import SessionLocal

router = APIRouter(
    prefix="/todos"
)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@router.post("", status_code=status.HTTP_201_CREATED)
def create_todo(todo: schemas.ToDoRequest, db: Session = Depends(get_db)):
    todo = crud.create_todo(db, todo)
    return todo

@router.get("", response_model=List[schemas.ToDoResponse])
def get_todos(completed: bool = None, db: Session = Depends(get_db)):
    todos = crud.read_todos(db, completed)
    return todos

@router.get("/{id}")
def get_todo_by_id(id: int, db: Session = Depends(get_db)):
    todo = crud.read_todo(db, id)
    if todo is None:
        raise HTTPException(status_code=404, detail="to do not found")
    return todo

@router.put("/{id}")
def update_todo(id: int, todo: schemas.ToDoRequest, db: Session = Depends(get_db)):
    todo = crud.update_todo(db, id, todo)
    if todo is None:
        raise HTTPException(status_code=404, detail="to do not found")
    return todo

@router.delete("/{id}", status_code=status.HTTP_200_OK)
def delete_todo(id: int, db: Session = Depends(get_db)):
    res = crud.delete_todo(db, id)
    if res is None:
        raise HTTPException(status_code=404, detail="to do not found")
```

## Testing

```python title="test_main.py"
from fastapi.testclient import TestClient

from main import app
from routers import todos


client = TestClient(app)


def test_read_main():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == "Hello World"
```


# FastAPI JWT API

- GitHub: https://github.com/travisluong/fullstackbook-jwt-fastapi
- YouTube: [Full Stack FastAPI + NextJS JWT Authentication Tutorial](https://youtu.be/InzrcSk_9YU)

## Command Line

```bash title="Terminal"
pip install fastapi "uvicorn[standard]" alembic pyscopg2 "python-jose[cryptography]" "passlib[bcrypt]" python-multipart
pip freeze > requirements.txt
alembic init alembic
alembic revision -m "create users table"
alembic revision -m "create roles table"
alembic revision -m "create users_to_roles table"
alembic upgrade head
uvicorn main:app --reload
```

## Database Migration

```python title="alembic/env.py"
import os
from logging.config import fileConfig

from sqlalchemy import engine_from_config
from sqlalchemy import pool

from alembic import context
from dotenv import load_dotenv

load_dotenv()

# this is the Alembic Config object, which provides
# access to the values within the .ini file in use.
config = context.config

config.set_main_option('sqlalchemy.url', f"postgresql://{os.environ['DATABASE_USER']}:@{os.environ['DATABASE_HOST']}/{os.environ['DATABASE_NAME']}")

# Interpret the config file for Python logging.
# This line sets up loggers basically.
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# add your model's MetaData object here
# for 'autogenerate' support
# from myapp import mymodel
# target_metadata = mymodel.Base.metadata
target_metadata = None

# other values from the config, defined by the needs of env.py,
# can be acquired:
# my_important_option = config.get_main_option("my_important_option")
# ... etc.


def run_migrations_offline() -> None:
    """Run migrations in 'offline' mode.

    This configures the context with just a URL
    and not an Engine, though an Engine is acceptable
    here as well.  By skipping the Engine creation
    we don't even need a DBAPI to be available.

    Calls to context.execute() here emit the given string to the
    script output.

    """
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    """Run migrations in 'online' mode.

    In this scenario we need to create an Engine
    and associate a connection with the context.

    """
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection, target_metadata=target_metadata
        )

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()

```

```sql title="alembic/versions/2d8360fae38a_create_users_to_roles_table.py"
"""create users_to_roles table

Revision ID: 2d8360fae38a
Revises: 9ce62b651a8f
Create Date: 2022-08-31 21:02:25.266259

"""
from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision = '2d8360fae38a'
down_revision = '9ce62b651a8f'
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.execute("""
    create table users_to_roles (
        user_id int references users (id),
        role_id int references roles (id)
    );
    """)


def downgrade() -> None:
    op.execute("drop table users_to_roles;")

```

```sql title="alembic/versions/9ce62b651a8f_create_roles_table.py"
"""create roles table

Revision ID: 9ce62b651a8f
Revises: 481b1e04c901
Create Date: 2022-08-31 21:02:10.601736

"""
from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision = '9ce62b651a8f'
down_revision = '481b1e04c901'
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.execute("""
    create table roles (
        id serial primary key,
        name text not null unique
    );
    """)


def downgrade() -> None:
    op.execute("drop table roles;")

```

```sql title="alembic/versions/481b1e04c901_create_users_table.py"
"""create users table

Revision ID: 481b1e04c901
Revises: 
Create Date: 2022-08-30 20:56:35.188712

"""
from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision = '481b1e04c901'
down_revision = None
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.execute("""
    create table users (
        id serial primary key,
        username text not null unique,
        email text not null unique,
        password text
    );
    """)


def downgrade() -> None:
    op.execute("drop table users;")

```

```sql title="alembic/versions/ece871602084_insert_roles.py"
"""insert roles

Revision ID: ece871602084
Revises: 2d8360fae38a
Create Date: 2022-08-31 21:05:14.000853

"""
from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision = 'ece871602084'
down_revision = '2d8360fae38a'
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.execute("""
    insert into roles (name) values ('ROLE_USER');
    insert into roles (name) values ('ROLE_MODERATOR');
    insert into roles (name) values ('ROLE_ADMIN');
    """)


def downgrade() -> None:
    op.execute("""
    delete from roles where name in ('ROLE_USER', 'ROLE_MODERATOR', 'ROLE_ADMIN');
    """)

```

## Dependencies

```python title="dependencies/auth.py"
from typing import Optional
from fastapi import Depends, HTTPException, status
from sqlalchemy.orm import Session
from datetime import datetime, timedelta
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt
from passlib.context import CryptContext

from database import engine
from model import User
from schemas import TokenData

# to get a string like this run:
# openssl rand -hex 32
SECRET_KEY = "09d25e094faa6ca2556c818166b7a9563b93f7099f6f0f4caa6cf63b88e8d3e7"
ALGORITHM = "HS256"

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/auth/token")

def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password):
    return pwd_context.hash(password)

def authenticate_user(username: str, password: str):
    with Session(engine) as session:
        results = session.query(User).filter(User.username == username)
        user = results.first()

        if not user:
            return False
        if not verify_password(password, user.password):
            return False
        return user

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

def get_current_user(token: str = Depends(oauth2_scheme)):
    with Session(engine) as session:
        credentials_exception = HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Could not validate credentials",
            headers={"WWW-Authenticate": "Bearer"},
        )
        try:
            payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
            username: str = payload.get("sub")
            if username is None:
                raise credentials_exception
            token_data = TokenData(username=username)
        except JWTError:
            raise credentials_exception
        results = session.query(User).filter(User.username == token_data.username)
        user = results.first()
        if user is None:
            raise credentials_exception
        return user

def get_current_active_user(current_user: User = Depends(get_current_user)):
    return current_user
```

```python title="dependencies/roles.py"
from sqlalchemy.orm import Session
from fastapi import Depends, HTTPException

from database import get_session
from model import Role, User, UserToRole
from routers.auth import get_current_active_user

def admin_role(session: Session = Depends(get_session), current_user: User = Depends(get_current_active_user)):
    roles = session.query(Role).filter(Role.name.in_(['ROLE_ADMIN']))
    role_ids = [role.id for role in roles]
    user_to_roles = session.query(UserToRole).filter(UserToRole.user_id == current_user.id).filter(UserToRole.role_id.in_(role_ids)).all()
    if len(user_to_roles) == 0:
        raise HTTPException(403, "Unauthorized.")
    return current_user

def moderator_role(session: Session = Depends(get_session), current_user: User = Depends(get_current_active_user)):
    roles = session.query(Role).filter(Role.name.in_(['ROLE_MODERATOR']))
    role_ids = [role.id for role in roles]
    user_to_roles = session.query(UserToRole).filter(UserToRole.user_id == current_user.id).filter(UserToRole.role_id.in_(role_ids)).all()
    if len(user_to_roles) == 0:
        raise HTTPException(403, "Unauthorized.")
    return current_user

def user_role(session: Session = Depends(get_session), current_user: User = Depends(get_current_active_user)):
    roles = session.query(Role).filter(Role.name.in_(['ROLE_USER', 'ROLE_MODERATOR', 'ROLE_ADMIN']))
    role_ids = [role.id for role in roles]
    user_to_roles = session.query(UserToRole).filter(UserToRole.user_id == current_user.id).filter(UserToRole.role_id.in_(role_ids)).all()
    if len(user_to_roles) == 0:
        raise HTTPException(403, "Unauthorized.")
    return current_user
```

## Routers

```python title="routers/auth.py"
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from datetime import timedelta
from fastapi.security import OAuth2PasswordRequestForm

from database import get_session
from dependencies.auth import authenticate_user, create_access_token, get_current_active_user, get_password_hash
from model import User, Role, UserToRole
from schemas import SignInRequest, Token, TokenJson, UserIn, UserRead

router = APIRouter(prefix="/api/auth")

ACCESS_TOKEN_EXPIRE_MINUTES = 43200

@router.post("/token", response_model=Token)
def login_for_access_token(
    form_data: OAuth2PasswordRequestForm = Depends()
):
    user = authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user.username}, expires_delta=access_token_expires
    )
    return {"access_token": access_token, "token_type": "bearer"}

@router.post("/signin", response_model=TokenJson)
def login_for_access_token(
    signin_request: SignInRequest
):
    user = authenticate_user(signin_request.username, signin_request.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user.username}, expires_delta=access_token_expires
    )
    return {"token": access_token, "token_type": "bearer"}

@router.get("/users/me", response_model=UserRead)
def read_users_me(current_user: User = Depends(get_current_active_user)):
    user = UserRead(username=current_user.username, email=current_user.email)
    return user

@router.get("/users/me/items")
def read_own_items(current_user: User = Depends(get_current_active_user)):
    return [{"item_id": "Foo", "owner": current_user.username}]

@router.post("/signup", response_model=UserRead)
def sign_up(user: UserIn, session: Session = Depends(get_session)):
    hashed_password = get_password_hash(user.password)
    role = session.query(Role).filter(Role.name == "ROLE_USER").first()
    user_to_role = UserToRole()
    user_to_role.role = role
    new_user = User(email=user.email, username=user.username, password=hashed_password)
    new_user.roles.append(user_to_role)
    session.add(new_user)
    session.commit()
    session.refresh(new_user)
    return UserRead(username=new_user.username, email=new_user.email)

```

## Configuration

```python title=".env.example"
DATABASE_HOST=localhost
DATABASE_NAME=fullstackbook-jwt-fastapi
DATABASE_USER=postgres
DATABASE_PASSWORD=
DATABASE_PORT=5432
APP_NAME="Full Stack Book JWT"
```

```python title="config.py"
from pydantic import BaseSettings

class Settings(BaseSettings):
    app_name: str = "Awesome API"

    class Config:
        env_file = ".env"
```

## Database

```python title="database.py"
import os
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from dotenv import load_dotenv

load_dotenv()

SQLALCHEMY_DATABASE_URL = f"postgresql://{os.environ['DATABASE_USER']}:@{os.environ['DATABASE_HOST']}/{os.environ['DATABASE_NAME']}"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL
)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()

def get_session():
    with Session(engine) as session:
        yield session
```

## Main

```python title="main.py"
from functools import lru_cache

from fastapi import FastAPI, Depends
from fastapi.responses import PlainTextResponse
from starlette.exceptions import HTTPException as StarletteHTTPException
from fastapi.middleware.cors import CORSMiddleware

from routers import auth, content

import config

app = FastAPI()
app.include_router(auth.router)
app.include_router(content.router)

origins = [
    "http://localhost:3000",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.exception_handler(StarletteHTTPException)
async def http_exception_handler(request, exc):
    print(f"{repr(exc)}")
    return PlainTextResponse(str(exc.detail), status_code=exc.status_code)

@lru_cache()
def get_settings():
    return config.Settings()

@app.get("/")
def read_root(settings: config.Settings = Depends(get_settings)):
    print(settings.app_name)
    return "Hello World"

```

## Model

```python title="model.py"
from sqlalchemy import Boolean, Column, ForeignKey, Integer, String
from sqlalchemy.orm import relationship

from database import Base

class UserToRole(Base):
    __tablename__ = "users_to_roles"
    
    user_id = Column(ForeignKey("users.id"), primary_key=True)
    role_id = Column(ForeignKey("roles.id"), primary_key=True)
    user = relationship("User", back_populates="roles")
    role = relationship("Role", back_populates="users")

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String)
    email = Column(String)
    password = Column(String)
    roles = relationship("UserToRole", back_populates="user")

class Role(Base):
    __tablename__ = "roles"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String)
    users = relationship("UserToRole", back_populates="role")


```

## Schemas

```python title="schemas.py"
from typing import Optional
from pydantic import BaseModel

class SignInRequest(BaseModel):
    username: str
    password: str

class Token(BaseModel):
    access_token: str
    token_type: str

class TokenJson(BaseModel):
    token: str
    token_type: str

class TokenData(BaseModel):
    username: Optional[str] = None

class UserIn(BaseModel):
    username: str
    email: str
    password: str

class UserRead(BaseModel):
    username: str
    email: str
```
