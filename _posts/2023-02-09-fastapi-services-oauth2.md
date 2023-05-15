---
layout: post
title: "Structuring FastAPI applications with multiple service logic, Postgres integration, and OAuth2 password authentication."
slug: fastapi-oauth2-postgres
description: Learn how to effectively structure a FastAPI application with multiple services, integrate with a Postgres backend, implement OAuth2 password authentication.
keywords: fastapi oauth2 jwt postgres
---

This tutorial provides an approach on how to effectively structure a FastAPI application
simplifying the implementation of multiple service logic, integrate it with Postgres backend, 
and implement straightforward OAuth2 Password authentication using Bearer and JWT tokens. 

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
that is somewhat similar to the three-tier architecture pattern. 

In this structure, the `routers` package serves as the user interface (UI) interaction layer. 
Each service comprises two components: (1) an application processing layer, implemented 
as a subclass of the `AppService` class, and (2) a data processing layer, implemented 
as a subclass of the `AppCRUD` class.

The `models` package provides SQLAlchemy mappings that establish the relationship between 
database objects and Python classes, while the `schemas` package represents serialized 
data models (Pydantic models) that are used throughout the application and as response objects.

The `backend` package provides a database session manager and application configuration
class. In scenarios where the application interacts with not only a database but also 
other backends, such as additional APIs, the respective clients can be placed within 
the `backend` package.

```bash
    .
    └── app/
        ├── backend             # Backend functionality and configs
        |   ├── config.py           # Configuration settings
        │   └── database.py         # Database session manager
        ├── models              # SQLAlchemy models
        │   ├── auth.py             # Authentication models
        |   ├── base.py             # Base classes, mixins
        |   └── ...                 # Service models
        ├── routers             # API routes
        |   ├── auth.py             # Authentication routers
        │   └── ...                 # Service routers
        ├── schemas             # Pydantic models
        |   ├── auth.py              
        │   └── ...
        ├── services            # Business logic
        |   ├── auth.py             # Create user, generate and verify tokens
        |   ├── base.py             # Base classes, mixins
        │   └── ...
        ├── cli.py              # Command-line utilities
        ├── const.py            # Constants
        ├── exc.py              # Exception handlers
        └── main.py             # Application runner
```

The `cli` module provides command-line functionality that is associated with API services but
does not require access through API endpoints. It contains commands that can be executed
from the command line to perform specific tasks, i.e. data manipulation, database operations, etc.

Module `main` represents FastAPI entry point and initiates `app` object (instance of `FastAPI` class).
The `app` object is then referred by server when running `uvicorn main:app` command.

## Adding a new service

To illustrate the approach, we will create a basic service that retrieves data from a 
Postgres backend and returns it to the user. 

### Backend setup

First, we create a database schema named `myapi` and create a table called `movies`. In this table,
we insert a list of records with following fields: `movie_id`, `title`, `released` (release year) and 
`rating` (e.g. imdb rating).

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
from sqlalchemy.orm import (
    Mapped,
    mapped_column,
)

from app.models.base import SQLModel


class MovieModel(SQLModel):
    __tablename__ = "movies"
    __table_args__ = {
        "schema": "myapi",
    }

    movie_id: Mapped[int] = mapped_column("movie_id", primary_key=True)
    title: Mapped[str] = mapped_column("title")
    released: Mapped[int] = mapped_column("released")
    rating: Mapped[float] = mapped_column("rating")
```

### Schemas

The `schemas` package provides Pydantic models that are used as an intermediary layer between
the source data (SQLAlchemy models) and the application output. Additionally, they are
used as the response objects.

Moving forward, we create a new file named `schemas/movies.py`. Within this file, we declare 
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

Package `routers` enables to define path operations and keep it organized, i.e. 
separate paths related to multiple services. As usual, let's create a new file for our
service `routers/movies.py` and define there two entry points: `get_movie` that returns
the movie given `movie_id`, and `get_new_movies` that implements selection with filtering
by release year and movie rating.

```python
from typing import List

from fastapi import (
    APIRouter,
    Depends,
)
from sqlalchemy.orm import Session

from app.backend.database import create_session
from app.schemas.movies import MovieSchema
from app.services.movies import MovieService

router = APIRouter(prefix="/movies")


@router.get("/", response_model=MovieSchema)
async def get_movie(
    movie_id: int,
    session: Session = Depends(create_session),
) -> MovieSchema:
    return MovieService(session).get_movie(movie_id)


@router.get("/new", response_model=List[MovieSchema])
async def get_new_movies(
    year: int,
    rating: float,
    session: Session = Depends(create_session),
) -> List[MovieSchema]:
    return MovieService(session).get_new_movies(year, rating)
```

### Services

As a last step, we create a new file `services/movies.py` where we implement service
related logic, in our case it's simply reading data from corresponding database objects and
converting it to the response schema.

Every service is a subclass of `AppService` class which provides database session object.

Data access methods are isolated from service logic as a subclass of `AppCRUD` class 
which provides helper functions for CRUD operations over database objects.

```python
from typing import List

from sqlalchemy import select

from app.models.movies import MovieModel
from app.schemas.movies import MovieSchema
from app.services.base import (
    AppCRUD,
    AppService,
)


class MovieService(AppService):
    def get_movie(self, movie_id: int) -> MovieSchema:
        return MovieCRUD(self.db).get_movie(movie_id)

    def get_movies(self, year: int, rating: float) -> List[MovieSchema]:
        return MovieCRUD(self.db).get_movies(year, rating)


class MovieCRUD(AppCRUD):
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

Configuration settings are provided via `backend/config.py` module and can be read from
environment variables prefixed with `MYAPI_`. It also supports dotenv parsing from `.env`
file placed in project root directory. 

```bash
$ cat .env
# Database DSN
MYAPI_DATABASE__DSN="postgresql://user:password@host:port/dbname"

# Token key for generating signatures
MYAPI_TOKEN_KEY="my_secret_key"
```

Or you can initialize everything using environment variables only.

```bash
$ MYAPI_DATABASE__DSN="postgresql://..." uvicorn app.main:app
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
```

## Authentication

Authentication service is integrated to application using the same design pattern
we used for adding `movies` service.

For demonstration purposes, we use OAuth2 Password grant type as a protocol to get an 
access token given `username` and `password`. The Password grant type is one of the
simplest OAuth grants and involves only one step: the app provides a login form to collect
user's credentials (username and password) and makes a POST request to the server in order
to exchange the password for an access token.

Note, that the Password grant is not recommended way as it requires the application collect
user's password. Check the OAuth 2.0 security best practices to remove the Password grant
from OAuth.

### Backend setup

Let's create table `users` in the backend database to store user information. 

```sql
CREATE TABLE IF NOT EXISTS myapi.users (
    user_id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL,
    hashed_password TEXT NOT NULL,
    UNIQUE(email)
);
```

Application does not store user's plain password, it uses hashing algorithm to encode 
passwords before writing data to database.

We add `auth` module in each of four principal packages (the same as we did for `movies` service).
`AuthService` class (see `services/auth.py` for details) below implements password hashing
and adding user data to database table.

```python
from passlib.context import CryptContext

from app.models.auth import UserModel
from app.schemas.auth import CreateUserSchema
from app.services.base import (
    AppCRUD,
    AppService,
)


pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


class HashingMixin:
    @staticmethod
    def bcrypt(password: str) -> str:
        return pwd_context.hash(password)

    @staticmethod
    def verify(hashed_password: str, plain_password: str) -> bool:
        return pwd_context.verify(plain_password, hashed_password)


class AuthService(HashingMixin, AppService):
    def create_user(self, user: CreateUserSchema) -> None:
        user_model = UserModel(
            name=user.name, email=user.email, hashed_password=self.bcrypt(user.password)
        )
        AuthCRUD(self.db).add_user(user_model)


class AuthCRUD(AppCRUD):
    def add_user(self, user: UserModel) -> None:
        self.add_one(user)
```

The convenience methods for making CRUD operations over users can be added to command 
line interface via `cli` module. Below is an example on how to create new user from 
command-line.

```python
import click

from app.backend.database import create_session
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

Executing `create-user` command from command-line, the new record will be added to `users` table.

```bash
myapi --name 'test user' --email test_user@myapi.com --password qwerty123
```

### Generating token

Application obtains `username` and `password` sent by user through `OAuth2PasswordRequestForm` 
form body via authentication endpoint. It extracts user information stored in database 
and verifies hashed password with the plain password obtained from request.

If verification succeeds, application generates temporary token and sends it back via 
response model to the user.

To generate and verify JWT tokens, application uses `python-jose` library with recommended 
cryptographic backend `pyca/cryptography`. To handle this process, we create random secret 
key and pass it to `config` via environment variable `MYAPI_TOKEN_KEY` (can be set also 
in dotenv file).

Example of token generation can be seen below. Method `authenticate` of `AuthService`
class is triggered every time user sends request to authentication endpoint (see 
`routers/auth.py` for details).

```python
from jose import jwt
from fastapi import (
    Depends,
    status,
)
from fastapi.security import OAuth2PasswordRequestForm

from app.backend.config import config
from app.exc import raise_with_log
from app.schemas.auth import TokenSchema
from app.services.base import AppService


class AuthService(AppService):
    def authenticate(
        self, login: OAuth2PasswordRequestForm = Depends()
    ) -> TokenSchema | None:
        user = AuthCRUD(self.db).get_user(login.username)

        if user.hashed_password is None:
            raise_with_log(status.HTTP_401_UNAUTHORIZED, "Incorrect password")
        else:
            if not self.verify(user.hashed_password, login.password):
                raise_with_log(status.HTTP_401_UNAUTHORIZED, "Incorrect password")
            else:
                access_token = self._create_access_token(user.name, user.email)
                return TokenSchema(access_token=access_token, token_type=TOKEN_TYPE)
        return None
    
    def _create_access_token(self, name: str, email: str) -> str:
        payload = {
            "name": name,
            "sub": email,
            "expires_at": self._expiration_time(),
        }

        return jwt.encode(payload, config.token_key, algorithm="HS256")
```

### Verification

User obtains JWT token from application and uses it to sign the request. To verify
token, application extracts user information from decoded token and verifies user 
validity and expiration time. If user is invalid or token has expired, it sends
the corresponding error message, otherwise it's processing the request.

Function `get_current_user` is responsible for token verification. 

```python
from jose import (
    jwt,
    JWTError,
)
from fastapi import (
    Depends,
    status,
)
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
```

Function `get_current_user` should be added as a dependency to every path operation
function that requires authentication. For example, if we want to add authentication
to `movies` service, we need to modify service routers in the following way.

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

Now putting it all together, let's run the application on localhost and test how it works.

```bash
$ uvicorn app.main:app
INFO:     Started server process [673616]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
```

Server is running, and we can send the authentication request.

```bash
$ curl -X 'POST' 'http://127.0.0.1:8000/token' -d 'username=test_user@myapi.com&password=qwerty123'
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVz...","token_type":"bearer"}%  
```

Authentication succeeded and we received access token.

If we send request with incorrect login data, application will raise an error.

```bash
$ curl -X 'POST' 'http://127.0.0.1:8000/token' -d 'username=test_user@myapi.com&password=password'
{"detail":"Incorrect password"}%  
```

Now, let's generate request to `movies` service and sign it with the obtained token.

```bash
$ curl -X 'GET' 'http://127.0.0.1:8000/movies/new?year=1990&rating=9' -H 'Authorization: Bearer eyJhbGc...'
[{"movie_id":1,"title":"The Shawshank Redemption","released":1994,"rating":9.2},{"movie_id":3,"title":"The Dark Knight","released":2008,"rating":9.0}]%  
```

Alright, it works.

Please check the source code in the [repository][2] for more detail.

[2]: https://github.com/viktorsapozhok/fastapi-services-oauth2 "viktorsapozhok/fastapi-services-oauth2"
