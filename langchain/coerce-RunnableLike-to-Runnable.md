While reading through the `langchain_core.runnable.base` code, the ability to create [`Runnable` objects](https://github.com/langchain-ai/langchain/blob/ddaf9de169e629ab3c56a76b2228d7f67054ef04/libs/core/langchain_core/runnables/base.py#L103) out of [`RunnableLike` objects](https://github.com/langchain-ai/langchain/blob/ddaf9de169e629ab3c56a76b2228d7f67054ef04/libs/core/langchain_core/runnables/base.py#L4368) caught my eye.

---

### What is LangChain?

[LangChain is a framework for developing applications powered by language models.](https://github.com/langchain-ai/langchain/tree/master?tab=readme-ov-file#-what-is-langchain)

### What is `langchain_core`?

Before [the release of *LangChain 0.1*](https://blog.langchain.dev/the-new-langchain-architecture-langchain-core-v0-1-langchain-community-and-a-path-to-langchain-v0-1/), there was no `langchain_core` package. Instead, there was just a monolithic `langchain` package.

Since the 0.1 release, `langchain` consists of three packages:

| Package | Description |
| --- | --- |
| `langchain` | higher-level, cognitive architecture style components such as use-case specific chains, agents and retrieval algorithms |
| `langchain_core` | base abstractions for the higher-level components and runtime for the LangChain Expression Language |
| `langchain_community` | third party integrations |

---

### `Runnable` objects are functions

From one perspective, a `Runnable` is a function: input goes in and input comes out.

```bash
graph TB
subgraph A Runnable is a function
    input
    output
    Runnable
    input --> Runnable --> output
end
```

LangChain types `Runnable` objects in this way [too](https://github.com/langchain-ai/langchain/blob/ddaf9de169e629ab3c56a76b2228d7f67054ef04/libs/core/langchain_core/runnables/base.py#L103C1-L103C45):

```python
class Runnable(Generic[Input, Output], ABC):
    ...
```

---

### `Runnable` objects are protocols

But a [`Runnable`](https://api.python.langchain.com/en/stable/runnables/langchain_core.runnables.base.Runnable.html#langchain-core-runnables-base-runnable) is also a protocol in the LangChain ecosystem. This is an important perspective to keep in mind. `Runnable` objects are the lifeblood of the LangChain ecosystem.

From an ecosystem point of view, a `Runnable` [is a unit of work that can be invoked, batched, streamed, transformed and composed.](https://api.python.langchain.com/en/latest/runnables/langchain_core.runnables.base.Runnable.html)

In other words, `Runnable` objects expose methods that a) the LangChain ecosystem does work with and which b) client code can use to leverage aspects of the LangChain ecosystem.

Consequently, a `Runnable` object's methods can be used to modify how the core execution logic within the `Runnable` receives and outputs data, as well as how it behaves and intersects with the LangChain ecosystem. For example:

* using batch execution
    
* with retry policies
    
* with lifecycle listeners
    
* executed declaratively (the LangChain Expression Language)
    
* executed imperatively (called directly with `component.invoke(...)`)
    
* parsing the output
    

But I know I have only seen the tip of the iceberg.

---

### `RunnableLike` objects can become `Runnable` objects

If an object is `RunnableLike`, it can augmented by the `Runnable` protocol.

[As the typing definitions show](https://github.com/langchain-ai/langchain/blob/ddaf9de169e629ab3c56a76b2228d7f67054ef04/libs/core/langchain_core/runnables/base.py#L4368C1-L4375C2), `RunnableLike` objects must share the same input-output structure of `Runnable` objects:

```python
RunnableLike = Union[
    Runnable[Input, Output],
    Callable[[Input], Output],
    Callable[[Input], Awaitable[Output]],
    Callable[[Iterator[Input]], Iterator[Output]],
    Callable[[AsyncIterator[Input]], AsyncIterator[Output]],
    Mapping[str, Any],
]
```

To create a `Runnable` object from a `RunnableLike` object, LangChain's `cast` function is called. The LangChain codebase uses the metaphor ['coerce'](https://github.com/langchain-ai/langchain/blob/ddaf9de169e629ab3c56a76b2228d7f67054ef04/libs/core/langchain_core/runnables/base.py#L4378):

```python
def coerce_to_runnable(thing: RunnableLike) -> Runnable[Input, Output]:
    """Coerce a runnable-like object into a Runnable.

    Args:
        thing: A runnable-like object.

    Returns:
        A Runnable.
    """
    if isinstance(thing, Runnable):
        return thing
    elif inspect.isasyncgenfunction(thing) or inspect.isgeneratorfunction(thing):
        return RunnableGenerator(thing)
    elif callable(thing):
        return RunnableLambda(cast(Callable[[Input], Output], thing))
    elif isinstance(thing, dict):
        return cast(Runnable[Input, Output], RunnableParallel(thing))
    else:
        raise TypeError(
            f"Expected a Runnable, callable or dict."
            f"Instead got an unsupported type: {type(thing)}"
        )
```

For example, a `Callable` 'thing' with the correct input-output behaviours is a `RunnableLike`. Therefore, when coerced to a `RunnableLambda`, it is augmented with the `Runnable` protocol's toolkit and methods of use.

---

### Using the coerced `RunnableLike`

Take for example a simple `add_one` function. It can be made a `Runnable`:

```python
from langchain_core.runnables import RunnableLambda

def add_one(x: int) -> int:
    return x + 1

add_runnable = RunnableLambda(add_one)
```

Now I can mediate my use of `add_one` through the `Runnable` interface.

This means I can use the `Runnable` protocol to modify, invoke, compose, trigger hooks, *etc* on top of `add_one`'s core execution logic.

---

I can **call** it imperatively with an input:

```bash
>>> add_runnable.invoke(2)
3
```

---

I can **batch** call it with a list of inputs:

```bash
>>> add_runnable.batch([2, 3, 4])
[3, 4, 5]
```

---

I can add event **listeners** on start, end and error:

```bash
>>> (add_runnable
     .with_listeners(on_start=lambda _: print("Starting..."))
     .invoke(2))
Starting...
3
```

---

If my `Runnable` is something that can fail and I want it to try again on failure, I can add a **retry policy**.

Let's demonstrate a retry policy by decorating `add_one` so that it fails twice before succeeding:

```python
def fail_twice_before_success(func):
    attempts = 0

    def inner(*args, **kwargs):
        nonlocal attempts
        print(f"Attempt {attempts + 1}")

        if attempts < 2:
            attempts += 1
            raise Exception("Failed")
        
        attempts = 0
        print("Success!")
        return func(*args, **kwargs)

    return inner

fail_twice_before_adding_one = fail_twice_before_success(add_one)
```

Now let's create a `Runnable` with a retry policy:

```bash
>>> fragile_runnable = (RunnableLambd(fail_twice_before_adding_one)
                        .with_retry_policy(max_attempts=3))
>>> fragile_runnable.invoke(2)
Attempt 1
Attempt 2
Attempt 3
Success!
3
```

---

Because it is augmented by the `Runnable` interface, I can include `add_one` in the [LangChain Expression Language](https://api.python.langchain.com/en/latest/runnables/langchain_core.runnables.base.Runnable.html#lcel-and-composition), which is a declarative way to compose `Runnable` objects into a [chain](https://api.python.langchain.com/en/latest/langchain_api_reference.html#module-langchain.chains).

```bash
>>> (add_runnable | add_runnable).invoke(0)
2
```

---

I can **add configurations**, such as configuring it with a basic logger that hooks into LangChain's callback system:

```python
>>> handler = StdOutCallbackHandler()
>>> config = {"callbacks": [handler]}
>>> ((add_runnable | add_runnable)
    .with_config(config)
    .invoke(0))
> Entering new RunnableSequence chain...


> Entering new RunnableLambda chain...

> Finished chain.


> Entering new RunnableLambda chain...

> Finished chain.

> Finished chain.
5
```

---

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text"><a target="_blank" rel="noopener noreferrer nofollow" href="https://warrenmarkham.hashnode.dev/" style="pointer-events: none"><strong>Hello, I'm Warren</strong></a>. I've worked in an AWS Data Engineer role at Infosys, Australia. Previously, I was a Disability Support Worker. I'm interested in collaborative workflows and going deeper into TDD, automation and distributed systems.</div>
</div>

<div data-node-type="callout">
<div data-node-type="callout-emoji">📆</div>
<div data-node-type="callout-text">I am currently studying Python at <a target="_blank" rel="noopener noreferrer nofollow" href="https://holbertonschool.com.au/" style="pointer-events: none"><strong>Holberton School Australia</strong></a>.</div>
</div>

<div data-node-type="callout">
<div data-node-type="callout-emoji">🐴</div>
<div data-node-type="callout-text">"<a target="_blank" rel="noopener noreferrer nofollow" href="https://holbertonschool.com.au/" style="pointer-events: none"><strong>Holberton School Australia</strong></a> is a tech school that trains software engineers through a collaborative, project-based curriculum. Over the course of 9 months, our students learn how to walk, talk, and code like software engineers."</div>
</div>
