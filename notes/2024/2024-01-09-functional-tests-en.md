# Functional testing #fastapi #pytest

For example, we decided to keep unit tests very fast, ready for pre-commit hooks or TDD. On the next level, we make functional tests enabling database and some external checks.
## Database

This is how we instantiate the database:

```python
async def get_database(
    settings: Settings = Depends(get_settings),
) -> Iterator[Database]:
	database = Database(
		database_url=settings.database_url,
	)
	yield database
	await database.close()
```

But now for fast tests we need some mock value. We can use `AsyncMock` to cover implementation by interfaces.

```python
async def get_database(
    settings: Settings = Depends(get_settings),
) -> Iterator[Union[Database, AsyncMock]]:
    if settings.database_url == SMOKE_VALUE:
        yield AsyncMock()
    else:
        database = Database(
            database_url=settings.database_url,
        )
        yield database
        await database.close()
```

The issue the nature of `AsyncMock` is very flexible and we easily can misspell any call without any complaints.

```python
if settings.database_url == SMOKE_VALUE:
    db_mock = create_autospec(Database)
    yield db_mock
```

We might user `create_autospec` from `unittest`. `create_autospec` https://docs.python.org/3/library/unittest.mock.html#create-autospec

For now we finish on this step, because mock topic of Python is very wide and worth a separate ~~book~~ note.

```python
SMOKE_VALUE = '<smoke>'

def _get_init_settings() -> dict[str, str]:
    init_settings: dict[str, str] = {}
    smoke_items = []

	# Run fast tests
    if os.getenv('RUN_SMOKE_TEST'):
        smoke_items = (
            'database_url',
        )
        init_settings = {item: SMOKE_VALUE for item in smoke_items}
	# Run functional tests
    elif os.getenv('RUN_FUNCTIONAL_TEST'):
        smoke_items = []
    init_settings = {item: SMOKE_VALUE for item in smoke_items}
    return init_settings


def get_settings() -> Settings:
	return Settings(**_get_init_settings())
```

`conftest.py`:
```python
import pytest

from fastapi.testclient import TestClient
from app import app

@pytest.fixture
def client() -> TestClient:
    yield TestClient(app)
```

## External auth systems

We have authentification with Auth0.
We have a digital code endpoint verification integrated with Auth0.

```python
from auth.auth import Auth0, AuthToken, BaseAuth
from fastapi import APIRouter, Depends, status
from fastapi.responses import JSONResponse
from inject import get_auth

router = APIRouter()


class CodeVerifyRequest(BaseModel):
    email: EmailStr
    code: constr(regex=r'^\d{6}$')


@router.post('/submit')
async def submit_code(
    code_verify_request: CodeVerifyRequest,
    auth: Auth0 = Depends(get_auth),
):
    token: AuthToken = await auth.verify_otp(
        email=code_verify_request.email,
        otp=code_verify_request.code,
    )
    return JSONResponse(
        content={
            'success': True,
            'access_token': token.access_token,
        },
        status_code=status.HTTP_202_ACCEPTED,
    )

```

The tricky part is where we choose what object to initiate: dummy for functional testing or production-ready for local testing.

```python
async def get_auth(
    settings: Settings = Depends(get_settings),
) -> Iterator[BaseAuth]:
    if settings.auth0_domain == SMOKE_VALUE:
        auth0 = BaseAuth()
    else:
        auth0 = Auth0(
            domain=settings.auth0_domain,
            client_id=settings.auth0_client_id,
            client_secret=settings.auth0_client_secret,
            connection=settings.auth0_connection,
            audience=settings.auth0_audience,
        )
    yield auth0
```

Overriding settings for different flows. Fast tests `RUN_SMOKE_TEST` and functional tests `RUN_FUNCTIONAL_TEST`:

```python
SMOKE_VALUE = '<smoke>'

def _get_init_settings() -> dict[str, str]:
    init_settings: dict[str, str] = {}
    smoke_items = []

	# Run fast tests
    if os.getenv('RUN_SMOKE_TEST'):
        smoke_items = [
            'database_url',
            'auth0_domain',
            'auth0_client_id', 
            'auth0_client_secret',
            'auth0_connection',
            'auth0_audience',
        ]
        init_settings = {item: SMOKE_VALUE for item in smoke_items}
	# Run functional tests
    elif os.getenv('RUN_FUNCTIONAL_TEST'):
        smoke_items = []
    init_settings = {item: SMOKE_VALUE for item in smoke_items}
    return init_settings


def get_settings() -> Settings:
	return Settings(**_get_init_settings())

```

For explicit running part of tests across we use own marker `@pytest.mark.require_db`

pytest markers settings might be placed in `pyproject.toml` [Link](https://docs.pytest.org/en/stable/reference/customize.html#pyproject-toml)

```toml
[tool.pytest.ini_options]
markers = [
    "require_db: marks tests as require database",
]
```

The usage of markers look like this:

```python
@pytest.mark.require_db
@pytest.mark.asyncio
async def test_code_verify_successful(
    client: TestClient,
    valid_code_verify_request: CodeVerifyRequest,
):
    pass
```

And we run it like this:

```bash
PYTHONPATH=. RUN_FUNCTIONAL_TEST=1 pytest -m "require_db"
```

This is a kind of duplication passing both environment settings and markers setting. This might be solved with pytest proper configuration, but this topic is skilled here.

Let's refer to model and our helper fixtures for the call:
```python
from pydantic import BaseModel, EmailStr, constr


class CodeVerifyRequest(BaseModel):
    email: EmailStr
    code: constr(regex=r'^\d{6}$')


@pytest.fixture
def valid_code_verify_request() -> CodeVerifyRequest:
    return CodeVerifyRequest(
        email='test@example.com',
        code='000000',
    )
```

Our final test is look like:

```python
@pytest.mark.require_db
@pytest.mark.asyncio
async def test_code_verify_successful(
    client: TestClient,
    valid_code_verify_request: CodeVerifyRequest,
):
    """Successful submit."""
    response = client.post(
        '/api/v1/verify-email/submit',
        json=valid_code_verify_request.dict(),
    )

    assert response.status_code == status.HTTP_202_ACCEPTED, response.text
    assert response.json()['success'], response.text
    assert response.json()['access_token'], response.text
```

This is how some unsuccessful test can look like:

```python
@pytest.mark.require_db
@pytest.mark.asyncio
async def test_empty_code_verify_error(
    client: TestClient,
    valid_code_verify_request: CodeVerifyRequest,
):
    valid_code_verify_request.code = ''

    response = client.post(
        '/api/v1/verify-email/submit',
        json=valid_code_verify_request.dict(),
    )

    assert (
        response.status_code == status.HTTP_422_UNPROCESSABLE_ENTITY
    ), response.text
    assert not response.json()['success'], response.text
    assert 'access_token' not in response.json(), response.text

```
# References

- `create_autospec` https://docs.python.org/3/library/unittest.mock.html#create-autospec
- pytest settings in pyproject.toml https://docs.pytest.org/en/stable/reference/customize.html#pyproject-toml