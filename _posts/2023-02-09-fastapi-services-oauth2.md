---
layout: post
title: "Structuring FastAPI application with multiple services using 3-tier design pattern."
slug: fastapi-oauth2-postgres
description: How to structure a FastAPI application using 3-tier design pattern, integrate with a Postgres backend via SQLAlchemy 2.0, implement OAuth2 password authentication.
keywords: fastapi sqlalchemy postgres 3-tier oauth2 jwt
---

This tutorial provides an approach on how to effectively structure a FastAPI application
with multiple services using 3-tier design pattern, integrate it with Postgres backend via SQLAlchemy 2.0, 
and implement straightforward OAuth2 Password authentication flow using Bearer and JSON Web Tokens (JWT). 

<br/>
<div class="blog-card">
<h7 class="m-1">source code repository: <a href="https://github.com/viktorsapozhok/fastapi-services-oauth2">viktorsapozhok/fastapi-services-oauth2</a></h7><br/><br/>
<a class="github-button" href="https://github.com/viktorsapozhok/fastapi-services-oauth2" data-icon="octicon-star" data-size="large" data-show-count="true" aria-label="Star viktorsapozhok/fastapi-services-oauth2 on GitHub">Star</a>
<a class="github-button" href="https://github.com/viktorsapozhok/fastapi-services-oauth2/fork" data-icon="octicon-repo-forked" data-size="large" data-show-count="true" aria-label="Fork viktorsapozhok/fastapi-services-oauth2 on GitHub">Fork</a>
<a class="github-button" href="https://github.com/viktorsapozhok" data-size="large" data-show-count="true" aria-label="Follow @viktorsapozhok on GitHub">Follow @viktorsapozhok</a>
</div>

## Structure overview

The application consists of four packages that offer service related functionality: 
`routers`, `services`, `schemas`, and `models`. To introduce a new service, it is necessary 
to add a new module within each of these packages. The proposed structure is designed in a manner 
that is somewhat similar to the [3-tier architecture pattern][1]. 

[1]: https://github.com/faif/python-patterns/blob/master/patterns/structural/3-tier.py "3-tier design pattern" 

```bash
    .
    └── app/
        ├── backend/            # Backend functionality and configs
        |   ├── config.py           # Configuration settings
        │   └── session.py          # Database session manager
        ├── models/             # SQLAlchemy models
        │   ├── auth.py             # Authentication models
        |   ├── base.py             # Base classes, mixins
        |   └── ...                 # Other service models
        ├── routers/            # API routes
        |   ├── auth.py             # Authentication routers
        │   └── ...                 # Other service routers
        ├── schemas/            # Pydantic models
        |   ├── auth.py              
        │   └── ...
        ├── services/           # Business logic
        |   ├── auth.py             # Create user, generate and verify tokens
        |   ├── base.py             # Base classes, mixins
        │   └── ...
        ├── cli.py              # Command-line utilities
        ├── const.py            # Constants
        ├── exc.py              # Exception handlers
        └── main.py             # Application runner
```

In this structure, the `routers` package serves as the user interface (UI) interaction layer. 
Each service comprises two components: (1) an application processing layer, implemented 
as a subclass of the `BaseService` class, and (2) a data processing layer, implemented 
as a subclass of the `BaseDataManager` class.

The `models` package provides SQLAlchemy mappings that establish the relationship between 
database objects and Python classes, while the `schemas` package represents serialized 
data models (Pydantic models) that are used throughout the application and as the response objects.

The `backend` package provides a database session manager and application configuration
class. In scenarios where the application interacts with not only a database but also 
other backends, such as additional APIs, the respective clients can be placed within 
the `backend` package.

The `cli` module provides command-line functionality that is associated with API services but
does not require access through API endpoints. It contains commands that can be executed
from the command line to perform specific tasks, i.e. data manipulation, database operations, etc.

Module `main` represents FastAPI entry point and initiates `app` object (instance of `FastAPI` class).
The `app` object is then referred by server when running `uvicorn main:app` command.

<a href="https://github.com/viktorsapozhok/fastapi-services-oauth2/blob/master/docs/source/images/3_tier.png?raw=true">
    <img 
        src="https://github.com/viktorsapozhok/fastapi-services-oauth2/blob/master/docs/source/images/3_tier.png?raw=true" 
        alt="Integrating FastAPI services using 3-tier design pattern"
    >
</a>

## Adding a new service

To illustrate the approach, we will create a basic service called `movies` that retrieves data from a 
Postgres backend and returns it to the user.

### Backend setup

First, we create a database schema named `myapi` and create a table called `movies`. In this table,
we insert a list of records with following fields: `movie_id`, `title`, `released` (release year) and 
`rating` (e.g. IMDB rating).

```sql
CREATE SCHEMA IF NOT EXISTS myapi;

CREATE TABLE IF NOT EXISTS myapi.movies (
    movie_id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    released INTEGER NOT NULL,
    rating NUMERIC(2, 1) NOT NULL
);
```

### Models

As the next step, we create a new file `models/movies.py` and declare there all the
SQLAlchemy models that are used within `movies` service. These models provide a mapping 
between the database objects and the corresponding Python classes.

```python
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import SQLModel


class MovieModel(SQLModel):
    __tablename__ = "movies"
    __table_args__ = {"schema": "myapi"}

    movie_id: Mapped[int] = mapped_column("movie_id", primary_key=True)
    title: Mapped[str] = mapped_column("title")
    released: Mapped[int] = mapped_column("released")
    rating: Mapped[float] = mapped_column("rating")
```

### Schemas

The `schemas` package provides Pydantic models that are used as an intermediary layer between
the source data (SQLAlchemy models) and the application output. Additionally, they are
used as the response objects.

Moving forward, we create a new file named `schemas/movies.py`. In this file, we declare 
all the schemas that are utilized within the `movies` service. In this specific case, 
we will have a single `MovieSchema` used as a request response model.

```python
from pydantic import BaseModel


class MovieSchema(BaseModel):
    movie_id: int
    title: str
    released: int
    rating: float
```

### Routers

The `routers` package enables to define path operations and keep it organized, 
allowing for the separation of paths associated with multiple services. 
We create a new file called `routers/movies.py`. In this file, we will define two entry points: 
`get_movie`, which retrieves the movie based on the provided `movie_id`, and `get_new_movies`, 
which implements a selection with applied filtering based on release year and movie rating.

```python
from typing import List

from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from app.backend.session import create_session
from app.schemas.movies import MovieSchema
from app.services.movies import MovieService

router = APIRouter(prefix="/movies")


@router.get("/", response_model=MovieSchema)
async def get_movie(
    movie_id: int, session: Session = Depends(create_session)
) -> MovieSchema:
    return MovieService(session).get_movie(movie_id)


@router.get("/new", response_model=List[MovieSchema])
async def get_new_movies(
    year: int, rating: float, session: Session = Depends(create_session)
) -> List[MovieSchema]:
    return MovieService(session).get_new_movies(year, rating)
```

### Services

As a final step, we proceed by creating a new file called `services/movies.py`. 
In this file, we will implement the specific logic associated with the `movies` 
service. In our case, this involves fetching data from the relevant database objects and 
transforming it into the desired response schemas.

Each service is implemented as a subclass of the `BaseService` class, which provides an
instance of the database session. This session can be delegated further down to the data 
processing layer.

To ensure separation of concerns, the data access methods are encapsulated within a subclass 
of the `BaseDataManager` class. This class offers convenience helpers for performing 
CRUD (Create, Read, Update, Delete) operations on the database objects, 
keeping them distinct from the main service logic.

```python
from typing import List

from sqlalchemy import select

from app.models.movies import MovieModel
from app.schemas.movies import MovieSchema
from app.services.base import BaseDataManager, BaseService


class MovieService(BaseService):
    def get_movie(self, movie_id: int) -> MovieSchema:
        return MovieDataManager(self.session).get_movie(movie_id)

    def get_movies(self, year: int, rating: float) -> List[MovieSchema]:
        return MovieDataManager(self.session).get_movies(year, rating)


class MovieDataManager(BaseDataManager):
    def get_movie(self, movie_id: int) -> MovieSchema:
        stmt = select(MovieModel).where(MovieModel.movie_id == movie_id)
        model = self.get_one(stmt)

        return MovieSchema(**model.to_dict())

    def get_movies(self, year: int, rating: float) -> List[MovieSchema]:
        schemas: List[MovieSchema] = list()

        stmt = select(MovieModel).where(
            MovieModel.released >= year,
            MovieModel.rating >= rating,
        )

        for model in self.get_all(stmt):
            schemas += [MovieSchema(**model.to_dict())]

        return schemas
```

### Config

Configuration settings are provided via `backend/config.py` module and can be obtained from
environment variables with the `MYAPI_` prefix. Additionally, it supports parsing of 
configuration settings from a `.env` file located in the project root directory.

```bash
$ cat .env
# Database DSN
MYAPI_DATABASE__DSN="postgresql://user:password@host:port/dbname"

# Token key for generating signatures
MYAPI_TOKEN_KEY="my_secret_key"
```

If you want to provide database DSN from environment variable then you can use following:

```bash
$ MYAPI_DATABASE__DSN="postgresql://..." uvicorn app.main:app
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
```

## Authentication

The authentication service is integrated into the application using the same design pattern
we used for adding the `movies` service. We place a file called `auth.py` within each of the 
four primary packages.

For demonstration purposes, the OAuth2 Password grant type is used as a protocol to obtain an 
access token based on the provided `username` and `password`. The Password grant type represents 
one of the simplest OAuth grants, involving a single step: the application provides a login 
form to collect the user's credentials (username and password) and initiates a POST request 
to the server to exchange the password for an access token.

Note, that while the Password grant is used in this tutorial for demonstration purposes, 
it is not a recommended approach as it requires the application to collect and handle the
user's password. Check the OAuth 2.0 security best practices to remove the Password grant
from the OAuth implementation.

Let's create a table called `users` in the database.

```sql
CREATE TABLE IF NOT EXISTS myapi.users (
    user_id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL,
    hashed_password TEXT NOT NULL,
    UNIQUE(email)
);
```

The application uses a hashing algorithm to encode passwords before storing them in the
database, ensuring that user passwords are not stored in plain text.

The `AuthService` class below (see `services/auth.py` for details) is implemented to handle 
password hashing and adding user data into the `users` table.

```python
from passlib.context import CryptContext

from app.models.auth import UserModel
from app.schemas.auth import CreateUserSchema
from app.services.base import BaseDataManager, BaseService

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


class HashingMixin:
    @staticmethod
    def bcrypt(password: str) -> str:
        return pwd_context.hash(password)

    @staticmethod
    def verify(hashed_password: str, plain_password: str) -> bool:
        return pwd_context.verify(plain_password, hashed_password)


class AuthService(HashingMixin, BaseService):
    def create_user(self, user: CreateUserSchema) -> None:
        user_model = UserModel(
            name=user.name, 
            email=user.email, 
            hashed_password=self.bcrypt(user.password),
        )
        AuthDataManager(self.session).add_user(user_model)


class AuthDataManager(BaseDataManager):
    def add_user(self, user: UserModel) -> None:
        self.add_one(user)
```

The convenience methods for performing CRUD operations on users can be incorporated using
the `cli` module. The following example demonstrates the process of creating a new user
from the command-line interface.

```python
import click

from app.backend.session import create_session
from app.schemas.auth import CreateUserSchema
from app.services.auth import AuthService


@click.group()
def main() -> None:
    pass


@main.command()
@click.option("--name", type=str, help="User name")
@click.option("--email", type=str, help="Email")
@click.option("--password", type=str, help="Password")
def create_user(name: str, email: str, password: str) -> None:
    user = CreateUserSchema(name=name, email=email, password=password)
    session = next(create_session())
    AuthService(session).create_user(user)
```

Now executing the `create-user` command from the command-line, a new record will be added to the `users` table.

```bash
$ myapi --name 'test user' --email test_user@myapi.com --password password
```

### Generating token

The application obtains `username` and `password` provided by the user through the 
`OAuth2PasswordRequestForm` form body, which is transmitted via authentication endpoint. 
It then extracts the user information stored in the database and verifies the hashed 
password against the plain password obtained from the request.

If verification succeeds, the application generates a temporary token and sends it back 
to the user via the response model.

To generate and verify JSON Web Tokens (JWT), the application utilizes the `python-jose` 
library with the recommended cryptographic backend, `pyca/cryptography`. To handle this process, 
a random secret key is generated and passed to the `config` module through the environment 
variable `MYAPI_TOKEN_KEY` (which also can be set in a dotenv file).

An example of token generation can be seen below. The `authenticate` method is triggered 
every time a user sends a request to the authentication endpoint. More details can be found
in the `routers/auth.py` module.

```python
from jose import jwt
from fastapi import Depends, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy import select

from app.backend.config import config
from app.exc import raise_with_log
from app.models.auth import UserModel
from app.schemas.auth import TokenSchema, UserSchema
from app.services.base import BaseDataManager, BaseService


class AuthService(BaseService):
    def authenticate(
        self, login: OAuth2PasswordRequestForm = Depends()
    ) -> TokenSchema | None:
        user = AuthDataManager(self.session).get_user(login.username)

        if user.hashed_password is None:
            raise_with_log(status.HTTP_401_UNAUTHORIZED, "Incorrect password")
        else:
            if not self.verify(user.hashed_password, login.password):
                raise_with_log(status.HTTP_401_UNAUTHORIZED, "Incorrect password")
            else:
                access_token = self._create_access_token(user.name, user.email)
                return TokenSchema(access_token=access_token, token_type="bearer")
        return None
    
    def _create_access_token(self, name: str, email: str) -> str:
        payload = {
            "name": name,
            "sub": email,
            "expires_at": self._expiration_time(),
        }

        return jwt.encode(payload, config.token_key, algorithm="HS256")


class AuthDataManager(BaseDataManager):
    def get_user(self, email: str) -> UserSchema:
        model = self.get_one(select(UserModel).where(UserModel.email == email))

        if not isinstance(model, UserModel):
            raise_with_log(status.HTTP_404_NOT_FOUND, "User not found")

        return UserSchema(
            name=model.name,
            email=model.email,
            hashed_password=model.hashed_password,
        )
```

### Verification

The user acquires a JWT token from the application and utilizes it to sign the request.
To verify it, the application extracts the user information from the decoded token,
and verifies both the validity of the user and the token expiration time. If verification
fails, the application sends a corresponding error message. Otherwise, it processes 
the request.

The `get_current_user` function is responsible for token verification. 

```python
from datetime import datetime

from jose import jwt, JWTError
from fastapi import Depends, status
from fastapi.security import OAuth2PasswordBearer

from app.backend.config import config
from app.exc import raise_with_log
from app.schemas.auth import UserSchema

oauth2_schema = OAuth2PasswordBearer(tokenUrl="token", auto_error=False)


async def get_current_user(token: str = Depends(oauth2_schema)):
    if not token:
        raise_with_log(status.HTTP_401_UNAUTHORIZED, "Invalid token")

    try:
        # decode token using secret token key provided by config
        payload = jwt.decode(token, config.token_key, algorithms="HS256")

        # extract encoded information
        name: int = payload.get("name")
        sub: str = payload.get("sub")
        expires_at: str = payload.get("expires_at")

        if sub is None:
            raise_with_log(status.HTTP_401_UNAUTHORIZED, "Invalid credentials")

        if is_expired(expires_at):
            raise_with_log(status.HTTP_401_UNAUTHORIZED, "Token expired")

        return UserSchema(name=name, email=sub)
    except JWTError:
        raise_with_log(status.HTTP_401_UNAUTHORIZED, "Invalid credentials")

    return None


def is_expired(expires_at: str) -> bool:
    return datetime.strptime(expires_at, "%Y-%m-%d %H:%M:%S") < datetime.utcnow()
```

The `get_current_user` function needs to be included as a dependency in each path operation
function that necessitates authentication. For instance, if we want to incorporate authentication
into the `movies` service, we must modify the service routers as shown below:

```python
from app.services.auth import get_current_user


@router.get("/", response_model=MovieSchema)
async def get_movie(
    movie_id: int,
    user: UserSchema = Depends(get_current_user),
    session: Session = Depends(create_session),
) -> MovieSchema:
    return MovieService(session).get_movie(movie_id)
```

## Running API

Now, putting it all together, let's run the application on the localhost and test how it works.

```bash
$ uvicorn app.main:app
INFO:     Started server process [673616]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
```

Server is running, and we can reach Swagger UI from the browser.

<a href="https://github.com/viktorsapozhok/fastapi-services-oauth2/blob/master/docs/source/images/swagger.png?raw=true">
    <img 
        src="https://github.com/viktorsapozhok/fastapi-services-oauth2/blob/master/docs/source/images/swagger.png?raw=true" 
        alt="FastAPI Swagger UI"
    >
</a>

Let's try to send the authentication request.

```bash
$ curl -X 'POST' 'http://127.0.0.1:8000/token' -d 'username=test_user@myapi.com&password=password'
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVz...","token_type":"bearer"}%  
```

Authentication succeeded, and we obtained an access token.

If we now attempt to send a request with incorrect login data, the application will raise an error.

```bash
$ curl -X 'POST' 'http://127.0.0.1:8000/token' -d 'username=test_user@myapi.com&password=qwerty'
{"detail":"Incorrect password"}%  
```

Now, let's generate a request to the `movies` service and sign it using the obtained token.

```bash
$ curl -X 'GET' 'http://127.0.0.1:8000/movies/new?year=1990&rating=9' -H 'Authorization: Bearer eyJhbGc...'
[{"movie_id":1,"title":"The Shawshank Redemption","released":1994,"rating":9.2},{"movie_id":3,"title":"The Dark Knight","released":2008,"rating":9.0}]%  
```

Alright, it works.

For more details, see source code in the [repository][2].

[2]: https://github.com/viktorsapozhok/fastapi-services-oauth2 "viktorsapozhok/fastapi-services-oauth2"
