## Functional testing #fastapi #pytest

```python
import logging

from auth.auth import AuthToken
from auth.auth0 import Auth0
from fastapi import APIRouter, Depends, status, Request
from fastapi.responses import JSONResponse
from inject import get_auth

router = APIRouter()


logger = logging.getLogger(__name__)


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
        client_ip=request.client.host,
    )

    return JSONResponse(
        content={
            'success': True,
            'access_token': token.access_token,
        },
        status_code=status.HTTP_202_ACCEPTED,
    )

```

Tricky part is where we choose what object to initiate: dummy for functional testing or production-ready for local testing.
```python
async def get_auth(
    settings: Settings = Depends(get_settings),
) -> Iterator[DummyLogicAuth0]:
    if settings.auth0_domain == SMOKE_VALUE:
        auth0 = DummyLogicAuth0()
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


```python
SMOKE_VALUE = '<smoke>'

def get_settings():
	if os.getenv('DO_FUNCTIONAL_TEST'):
        smoke_items = (
            'auth0_domain',
        )
        init_settings = {item: SMOKE_VALUE for item in smoke_items}
    return init_settings
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

```python
from pydantic import BaseModel, EmailStr, constr


class CodeVerifyRequest(BaseModel):
    email: EmailStr
    code: constr(regex=r'^\d{6}$')


@pytest.fixture
def valid_code_verify_request(mfield: Field) -> CodeVerifyRequest:
    return CodeVerifyRequest(
        email='test@example.com',
        code='000000',
    )


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


```python
from pydantic import BaseModel, EmailStr, constr


class CodeVerifyRequest(BaseModel):
    email: EmailStr
    code: constr(regex=r'^\d{6}$')


@pytest.fixture
def valid_code_verify_request(mfield: Field) -> CodeVerifyRequest:
    return CodeVerifyRequest(
        email=mfield('person.email'),
        code='000000',
    )


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