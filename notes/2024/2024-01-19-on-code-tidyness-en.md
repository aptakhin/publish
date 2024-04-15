# On code tidyness

We had multiple star imports from Python disregarding PEP8 recommendations. It was not the big issue from understanding, but linters and loading were more tricky.

```
pip install removestar
```

```bash
# We add some explicit common imports, not to get them imported from parent modules
find *.py -type f -print0 | xargs -0 -I {} sed -i '1s/^/from decimal import Decimal\nimport re\nfrom .settings import Settings\n\n/' {}

# Remove
find *.py -type f -print0 | xargs -0 -I {} removestar -i {} --max-line-length 79

# Remove unused imports
unimport ./

# Make sort
isort ./

# Remove first line if empty
find *.py -type f -print0 | xargs -o -I {} sed '1{/^$/d}' {}
```

`unimport` and `sort` also can be replaced by `ruff format`.

## Code smell metrics

ruff Mccabe
https://xenon.readthedocs.io/en/latest/
https://radon.readthedocs.io/en/latest/

https://github.com/sasanjac/flake8-cohesion
https://pypi.org/project/flake8-cognitive-complexity/
https://pypi.org/project/flake8-expression-complexity/
https://github.com/Miserlou/JonesComplexity

Nice explanation about some differences: https://github.com/astral-sh/ruff/issues/2418#issuecomment-1561661962