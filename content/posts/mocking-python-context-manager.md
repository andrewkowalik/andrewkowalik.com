---
title: "Mocking Python Context Managers"
date: 2016-04-15T05:30:54-08:00
draft: false
---

Mocking context managers, complex to start and obvious afterwards. Looking back at the example now I am amazed how confused I was. At the time it was a frustrating mess figuring out how to properly mock it, I was constantly mocking the incorrect code at runtime. My google-fu was also failing me as most of the examples were outdated or did not fully dive into what I was attempting to do. Here is my solution.

We have a barebones context manager that simply returns an instance of our database client.  The database client is abbrieviated but to illustrate I have included the changes to `__enter__` to demonstrate how the context manager works.
```python
# foobar.py
class DBClient(object):
    def __enter__(self):
        return self

def client_context_manager():
    return DBClient()

def get_item(id):
    with client_context_manager() as client:
        new_id = client.foobar()
        client.get_from_id(new_id)
```

For the purposes of this example lets assume we want to test client.foobar() raising an exception. To properly mock the context manager we need to first patch it. We then need to mock out the `DBClient`. Since the client class can act as a context manager we have to override `__enter__` to ensure that we are returning an instance of the mock and not of the actual client instance. This one stumped me for a bit. Lastly the context manager function returns this new mock we have created. 

```python
# test.py
@mock.patch('foobar.client_context_manager', autospec=True)
def some_test(self, patch):
    mock_client = mock.MagicMock(spec=DBClient)
    mock_client.get_from_id = mock.Mock()
    mock_client.foobar.side_effect = IndexError
    mock_client.__enter__.return_value = mock_client
    patch.return_value = mock_client
```

Would love to see if anyone has any recommendations on how to further improve this pattern.
