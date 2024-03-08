I spent some time looking at [`requests/src/requests/models.py`](https://github.com/psf/requests/blob/eea3bbf9ac635f465ee6c9903dc57c677952dafd/src/requests/models.py#L730).
## I learned, observed or thought about:
* `requests.Request` provides an example of [customising the Python Data Model](https://github.com/psf/requests/blob/eea3bbf9ac635f465ee6c9903dc57c677952dafd/src/requests/models.py#L730) - its behaviour in a Boolean context is changed by overriding `__bool__`.
    
* Some pattern ideas:
    
    1. [*Raise an exception to determine the value of a property*](https://github.com/psf/requests/blob/eea3bbf9ac635f465ee6c9903dc57c677952dafd/src/requests/models.py#L755).
        
    2. [*Encapsulate an exception trigger in a callable for on-demand delivery*](https://github.com/psf/requests/blob/eea3bbf9ac635f465ee6c9903dc57c677952dafd/src/requests/models.py#L764)*.*
        
    3. [*Raise an exception if able to build an error message*](https://github.com/psf/requests/blob/eea3bbf9ac635f465ee6c9903dc57c677952dafd/src/requests/models.py#L992).
        
    4. (More generally) *Attempt to build a string and act if non-empty*.
        
* `property` objects are data descriptors (although I confused myself at first).
    

---

### The meandering note-taking

#### Customise the Python Data model

This is one way of doing things with `requests.Response` objects:

```python
import requests

response = requests.get("https://api.github.com")
if response.status_code == 200:
    print("Success")
elif response.status_code = 404:
    print("Failure")
```

But `requests.Request` objects also have the following behaviour:

```python
>>> response = requests.get("https://api.github.com")
>>> response.status_code
200
>>> bool(response)
True
>>> response.status_code = 401
>>> bool(response)
False
```

That is, the truth value testing of a `requests.Request` object is based on the status code of the response.

> ## [Truth Value Testing](https://docs.python.org/3/library/stdtypes.html#truth-value-testing)
> 
> Any object can be tested for truth value, for use in an `if` or `while` condition or as operand of the Boolean operations below \[*note:*`or`, `and` and `not`\].

The behaviour of a `Requests.response` object in a Boolean context is an example of the customisable nature of Python's data model:

> By default, an object is considered true unless its class defines either a `__bool__()` method that returns False or a `__len__()` method that returns zero, when called with the object.

That is precisely what `requests.Response.__bool__` does. It customises the Data Model so that instances of `requests.Response` return `False` in a Boolean context if their status code is 400 or above. The default behaviour would have been to always return `True`.

```python
def __bool__(self):
        """Returns True if :attr:`status_code` is less than 400.

        This attribute checks if the status code of the response is between
        400 and 600 to see if there was a client error or a server error. If
        the status code, is between 200 and 400, this will return True. This
        is **not** a check to see if the response code is ``200 OK``.
        """
        return self.ok
```

#### Delegating the `__bool__` call to a dotted lookup process
Yet the plot also thickens. `__bool__` delegates to the dotted lookup process for the attribute `self.ok`. I imagine the runtime finds a descriptor object in the class dictionary of `requests.Response`, and that this descriptor object then calculates the value of `self.ok` by calling its `__get__` method and returning the result.

That is exactly what happens! Well, sort of. There *is* a non-data descriptor object in the class dictionary of `Response`, and we see how it doesn't play nice with attribute storage and deletion (because non-data descriptors do not implement the `__set__` and `__delete__` methods of the descriptor protocol):

```python
>>> type(response).__dict__['ok']
<property object at 0x7f75cac3c4f0>
>>> response.ok = 3
Traceback (most recent call last):
  ...
AttributeError: property 'ok' of 'Response' object has no deleter
>>> del reponse.ok
Traceback (most recent call last):
  ....
AttributeError: property 'ok' of 'Response' object has no deleter
```

*Edit: Bzzz - Wrong!* `@property` actually creates a *data* descriptor object, and it is precisely because `self.ok` interacts with a class dictionary object that is a descriptor that the above behaviour is observed.

**Wait! Did I just say something wrong?**

Let's back track that. But keep the kettle on the stove. I want to look at what I just said about the `property` object being a non-data descriptor.

Is`Response.ok` attribute a non-data descriptor like I just claimed (no, it is not)? 

If it is, then dotted lookup should a) find the descriptor object in the class dictionary but then b) see if there is anything in the instance dictionary before otherwise c) delegating attribute attribute storage, retrieval and deletion to the methods of the non-data descriptor.

But that is not what happens. The class attribute `ok` behaves like a data descriptor - attribute lookup does not storm ahead with the instance dictionary, like you would expect of a non-data descriptor. Instead, it still executes on the descriptor pathway:

```python
>>> response.__dict__['ok'] = 3
>>> response.ok
True
```

So is a `property` object a *data* descriptor then? Yes!

```bash
>>> [hasattr(property, attr) for attr in ['__get__', '__set__', '__delete__']]
[True, True, True]
```

And I *should* have known that from this behaviour alone:

```bash
>>> response.ok = 3
Traceback (most recent call last):
  ...
AttributeError: property 'ok' of 'Response' object has no deleter
>>> del reponse.ok
Traceback (most recent call last):
  ....
AttributeError: property 'ok' of 'Response' object has no deleter
```

I should have known because if the attribute `ok` was a nondata descriptor, then `response.ok = 3` would just have bound the name `ok` to the value `3` in the instance dictionary. No Python machinery would have interrupted the attribute behaviour.

Storage into the instance dictionary has priority over the actions of a non-data descriptor (which has no `__set__` method defined by definition anyway). And then `del response.ok` would just remove the name `ok` from the instance dictionary. Because, again, attribute deletion acts at the level of the instance dictionary before a non-data descriptor can act (which has no `__del__` method defined anyway, so there *is* no action it could take).

How a non-data descriptor would behave can be demonstrated:

```python
class NonDataDescriptor:
    def __get__(self, instance, owner):
        return True

class ResponseUsingNonDataDescriptor:
    ok = NonDataDescriptor()

response = ResponseUsingNonDataDescriptor()
response.ok = 3     # no AttributeError
del response.ok     # no AttributeError
```

But that isn't what happened. An `AttributeError` was raised. This implies that the Python runtime discovered something in its execution path that preempted interaction with the instance dictionary. Only the descriptor protocol has the necessary 'hooks' to interrupt the instance dictionary binding process: discovering a data descriptor in the class attribute and calling its `__set__` method instead.

Therefore, a `property` object is a data descriptor; it must just be that its implementation of `__set__` (and `__del__`) raises `AttributeError` exceptions.

That ends the back track. Continuing...

### Back to how `requests.Response` delegate truth value testing

Once inside the `ok` property code block, there is at least one more 'hop' before establishing a Boolean value for `requests.Response` objects. A call gets made to `self.raise_for_status()` to determine whether `self.ok` should return `True` or `False`.

```python
class Response:
    
    ...
    
    @property
    def ok(self):
        """Returns True if :attr:`status_code` is less than 400, False if not.

        This attribute checks if the status code of the response is between
        400 and 600 to see if there was a client error or a server error. If
        the status code is between 200 and 400, this will return True. This
        is **not** a check to see if the response code is ``200 OK``.
        """
        try:
            self.raise_for_status()
        except HTTPError:
            return False
        return True
```

So interesting! There are two patterns that catch my eye here:

1. *Raise an exception to determine the value of a property*.
    
2. *Encapsulate an exception trigger in a callable for on-demand delivery*.
    

Notably, in the `ok` property code block, the exception message is swallowed - a substantial reduction in how much information is communicated. But it doesn't *have* to be swallowed. `raise_for_status(self)` simply delivers or does not deliver a `HTTPError`; it is up to the calling environment to introspect the exception to whatever degree is needed for decision-making.

As for the `raise_for_status(self)` code itself, there is a pattern that intrigues me there too: *attempt to build a string then act only if the string is non-empty*.

```python
class Response:
    
    ...

    def raise_for_status(self):
        """Raises :class:`HTTPError`, if one occurred."""

        http_error_msg = ""
        if isinstance(self.reason, bytes):
            # We attempt to decode utf-8 first because some servers
            # choose to localize their reason strings. If the string
            # isn't utf-8, we fall back to iso-8859-1 for all other
            # encodings. (See PR #3538)
            try:
                reason = self.reason.decode("utf-8")
            except UnicodeDecodeError:
                reason = self.reason.decode("iso-8859-1")
        else:
            reason = self.reason

        if 400 <= self.status_code < 500:
            http_error_msg = (
                f"{self.status_code} Client Error: {reason} for url: {self.url}"
            )

        elif 500 <= self.status_code < 600:
            http_error_msg = (
                f"{self.status_code} Server Error: {reason} for url: {self.url}"
            )

        if http_error_msg:
            raise HTTPError(http_error_msg, response=self)
```

Or, in the language of the code block, the 'Attempt to build a string' pattern is:

* *Only raise an exception if able to build an error message*.
    

### Tests

*I like to write tests. They are experiential and teach me things. So I often practice the skill when note-taking.*

```python
import pytest

def test_that_Request_object_is_truthy_for_status_codes_below_401():
    response = requests.get("https://api.github.com")
    for status_code in range(100, 401):
        response.status_code = status_code
        assert bool(response) == True

def test_that_Request_object_is_falsey_for_status_codes_above_400():
    response = requests.get("https://api.github.com")
    for status_code in range(401, 600):
        response.status_code = status_code
        assert bool(response) == False
```

---

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text"><a target="_blank" rel="noopener noreferrer nofollow" href="https://warrenmarkham.hashnode.dev/" style="pointer-events: none"><strong>Hello, I'm Warren</strong></a>. I've worked in an AWS Data Engineer role at Infosys, Australia. Previously, I was a Disability Support Worker. I'm interested in collaborative workflows and going deeper into Python, AI, TDD, automation and distributed systems.</div>
</div>

<div data-node-type="callout">
<div data-node-type="callout-emoji">📆</div>
<div data-node-type="callout-text">I am currently studying Python at <a target="_blank" rel="noopener noreferrer nofollow" href="https://holbertonschool.com.au/" style="pointer-events: none"><strong>Holberton School Australia</strong></a>.</div>
</div>

<div data-node-type="callout">
<div data-node-type="callout-emoji">🐴</div>
<div data-node-type="callout-text">"<a target="_blank" rel="noopener noreferrer nofollow" href="https://holbertonschool.com.au/" style="pointer-events: none"><strong>Holberton School Australia</strong></a> is a tech school that trains software engineers through a collaborative, project-based curriculum. Over the course of 9 months, our students learn how to walk, talk, and code like software engineers."</div>
</div>
