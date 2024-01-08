## Unit-tests

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

Test one general case:
```python
def test_get_sas_link_file_extension():
    link = 'https://windows.net/documents/bank_statement%2Fd7305afa-6da8-46c8-bfae-fb47df2d7ab0.pdf?st=2024-01-05T'  # noqa: E501
    assert get_sas_link_file_extension(link) == 'pdf'
```

Generally it's fine, but we want to be sure our contract is valid:
```python
def test_get_sas_link_file_extension():
    link = 'https://windows.net/documents/bank_statement%2Fd7305afa-6da8-46c8-bfae-fb47df2d7ab0.pdf?st=2024-01-05T'  # noqa: E501
    assert get_sas_link_file_extension(link) == 'pdf'

    no_ext_link = 'https://windows.net/documents/bank_statement?st=2024-01-05T'
    assert get_sas_link_file_extension(no_ext_link) is None

	assert get_sas_link_file_extension(None) is None
```

No we are covered 100%, but sometime we are not sure that we checked everything. For advanced usage topic of mutation testing: 
- https://opensource.com/article/20/7/mutmut-python
- https://python.plainenglish.io/python-mutation-testing-with-cosmic-ray-4b78eb9e0676
