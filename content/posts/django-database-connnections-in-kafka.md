---
title: "Cleaning up Django Persistent Database Connections"
date: 2017-02-13T06:01:39-08:00
draft: true
---

For the past year at work we have been utilizing more Kafka infrastructure for what typically would have been handled using celery workers. This has led to some interesting rediscoveries of Django ORM. While this example is used in a Kafka implementation, it is not dependent on it and can be used in any scenario where code is run outside of a typical Django view.

We started noticing long running idle connections against MySQL. After tracing these back to our Kafka workers we soon realized that Django ORM deals with persistent database (DB) connections that are tightly coupled with the request/response cycle. The DB connections are released once a view generates the response. In the case of delayed workers, there is no request/response so Django ORM holds these connections open until MySQL forcefully closes them.

Since Celery is a gold standard for python delayed jobs and was born out of a Django implementation it must have already solved this problem. In fact it does a lot for trying to make Django/Django ORM work in the context of a worker process. [Django Pre/Post Task run cleanup](https://github.com/celery/celery/blob/master/celery/fixups/django.py#L163). Celery its a little more explicit which could be due to the age of the implementation. There is explicit cleanup that iterates through the list of available connections and closes them. In our case I think we can use some of Django's own healer functions to perform the cleanup.

Django has a wonderful helper [function](https://github.com/django/django/blob/master/django/db/__init__.py#L59), `close_old_connections`, which will iterate through the list of functions and close if unusable or obsolete. Each Django ORM DB connection has a configurable life. This code will close the connection if the connection life exceeds the allowed life.

Now all that is left is how to run this function in the context of a Kafka worker. In this example lets assume our Kafka work is setup similarly to a Celery unit of work. We have a function that represents the type of work, this means we can wrap the function using a decorator. 

```python
from functools import wraps

from django import db

def cleanup_db_connections(func):

    @wraps(func)
    def wrapper(*args, **kwargs):
        try:
            r_val = func(*args, **kwargs)
        except db.OperationalError as e:
            db.close_old_connections()
            r_val = func(*args, **kwargs)
        finally:
            db.close_old_connections()

        return r_val

    return wrapper
```

Wrapper behavior in order

- Attempt to run the wrapped function
- If Django ORM OperationalError is hit, close old connections and retry. This Error is raised in the case of a forcefully closed connection.
- Always close all old connections at the end of the function.
