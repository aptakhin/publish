# Unit-tests

## Some simple backend function #pytest

I wrote some code. Focused a bit more on clear naming not to repeat what does it do in docstring.

```python
from urllib.parse import urlparse

def get_sas_link_file_extension(url: str) -> Optional[str]:
    if not url:
        return None
    parse_result = urlparse(url)
    # I'm not sure about file-system methods for parsing urls
    # Will be old-school plain
    ext_split = parse_result.path.rsplit('.', 1)
    if len(ext_split) == 1:
        return None
    return ext_split[1]
```

What is missing? Outside the team might be no understanding what is `sas` in the function name. So, somewhere we need to give explanation that "A shared access signature (SAS) provides secure delegated access to resources in your storage account". And storage account is some S3-like storage in Azure. But ok, we are about tests.

Test one general case code is working:

```python
def test_get_sas_link_file_extension():
    link = 'https://windows.net/documents/bank_statement%2Fd7305afa-6da8-46c8-bfae-fb47df2d7ab0.pdf?st=2024-01-05T'  # noqa: E501
    assert get_sas_link_file_extension(link) == 'pdf'
```

Generally, it's fine; we don't have other different types of links. But we want to be sure our contract is valid:

```python
def test_get_sas_link_file_extension():
    link = 'https://windows.net/documents/bank_statement%2Fd7305afa-6da8-46c8-bfae-fb47df2d7ab0.pdf?st=2024-01-05T'  # noqa: E501
    assert get_sas_link_file_extension(link) == 'pdf'

    no_ext_link = 'https://windows.net/documents/bank_statement?st=2024-01-05T'
    assert get_sas_link_file_extension(no_ext_link) is None

	assert get_sas_link_file_extension(None) is None
```

What is a contract? It's some agreement on what arguments the code receiving is valid and which output is also valid. More formally I added link to wiki in the references in the botton.

For runtime ensuring in contracts in Python we can use `deal` library. Also, I added in the bottom.

Now, we are covered 100%, but sometime we are not sure that we checked everything. For advanced usage topic of mutation testing.

## Endpoints test #fastapi #pytest

`conftest.py`:
```python
import pytest

from fastapi.testclient import TestClient
from app import app

@pytest.fixture
def client() -> TestClient:
    yield TestClient(app)
```

This is a first easy test. Just check with TestClient that we get an okay healthcheck response.

```python
@pytest.mark.asyncio
async def test_health(client: TestClient):
    response = client.get('/api/v1/healthz')

    assert response.status_code == status.HTTP_200_OK
    assert response.json() == {'ok': True}, response.text
```

The second argument can be useful if we get assert fail and want to see the actual response.

```python
assert response.json() == {'ok': True}, response.text
```

`async` for this test is not really required, but it's for more general case. Also remember you will need plugin `pytest-asyncio` to support this. 
# References
- SAS-link explanation https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview
- Design by contract https://en.wikipedia.org/wiki/Design_by_contract
- `deal` library https://github.com/life4/deal?tab=readme-ov-file#deal-in-30-seconds
- Mutation testing 1 https://opensource.com/article/20/7/mutmut-python
- Mutation testing 2 https://python.plainenglish.io/python-mutation-testing-with-cosmic-ray-4b78eb9e0676
