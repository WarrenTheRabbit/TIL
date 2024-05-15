You can include the project root in pytest's Python import path using `pytest.ini` or `pyproject.toml`.


### Scenario 

 If you have
```bash
.
├── 0-functions.py
├── README.md
└── tests
    └── test_pagination.py
```

and you want to import `0-functions.py` into `test_pagination.py` using

```python
def test_index_range():
  index_range = __import__('0-functions').index_range
```

you get an error.

### Solution 1
Create a `pytest.ini` file in the root folder with the entry 

```
[pytest]
pythonpath = .
```

### Solution 2
Create a `pyproject.toml` file in the root folder with the entry

```
[tool.pytest.ini_options]
pythonpath = ["."]
```

