# pytest + FastAPI

`conftest.py`:
```python
import pytest 

from fastapi.testclient import TestClient
from app import app

@pytest.fixture
def client() -> TestClient:
    yield TestClient(app)
```

This is a first easy test. Just check that with TestClient we get ok healthcheck response.

```python
@pytest.mark.asyncio
async def test_health(client: TestClient):
    response = client.get('/api/v1/healthz')

    assert response.status_code == status.HTTP_200_OK
    assert response.json() == {'ok': True}, response.text
```

The second argument can be useful if we get assert fail and we want to see the actual response.

```python
assert response.json() == {'ok': True}, response.text
```

