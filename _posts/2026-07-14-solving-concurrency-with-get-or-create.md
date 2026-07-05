---
layout: post
title: Solving concurrency with get_or_create
date: 2026-07-14
categories: [django]
tags: [concurrency]
description: How Django's get_or_create handles race conditions under the hood.
---

Someone noticed that we were getting errors within our celery tasks when we were running batch jobs. I decided to take a
look and this is what I found.

## The Error

When looking at the error, I could see that the celery task was raising an `IntegrityError` via an `INSERT` statement.
This alone tells us that this failed when trying to create an entry vs just trying update it.

Here's a rough view of how the relevant code looks:

```python
# ...
try:
    Model.objects.get(...)
except:
    Model.objects.create(...)

if some_condition_that_is_only_true_when_model_already_existed:
    # Business logic that requires doing calculations and updating multiple properties.
    model.attribute = value
    model.save()
# ...
```

Now looking at this code, and knowing that its a celery task, we can make a pretty insightful guess that this was a
concurrency bug. Why? Well our batch tasks can take multiple ID's (some of which are duplicates) and if two are running
at the same time then both can fail at the `get` call, and then both will try to run the `create` call and by the
constraints that we have on the table, only of them will succeed and the other will fail.

## Solution

In situations like this, a simple solution is to wrap the `create` call in a transaction and if that fails, then retry
doing a `get` since it's most likely been created and we can fetch it now. Something like this:


```python
try:
    model =  Model.objects.get(id=id)
except Model.DoesNotExist:
    try:
        with transaction.atomic():
            model = Model.objects.create(...)
    except IntegrityError:
        model = Model.objects.get(id=id)
```

To test my theory out, I `rubber ducked` my thoughts against an LLM, and it agreed with my conclusion. It also suggested
to use Django's [`get_or_create`](https://docs.djangoproject.com/en/5.2/ref/models/querysets/#get-or-create) to solve
this problem which baffled me since I always viewed that method as nothing more than a convenience wrapper of the first
code snippet above. But after reading the documentation for it and viewing the source code to see whats actually
implemented, it ended up being the solution I exactly needed. For convenience here is the method as of Django 5.2:

```python
def get_or_create(self, defaults=None, **kwargs):
    """
    Look up an object with the given kwargs, creating one if necessary.
    Return a tuple of (object, created), where created is a boolean
    specifying whether an object was created.
    """
    # The get() needs to be targeted at the write database in order
    # to avoid potential transaction consistency problems.
    self._for_write = True
    try:
        return self.get(**kwargs), False
    except self.model.DoesNotExist:
        params = self._extract_model_params(defaults, **kwargs)
        # Try to create an object using passed params.
        try:
            with transaction.atomic(using=self.db):
                params = dict(resolve_callables(params))
                return self.create(**params), True
        except IntegrityError:
            try:
                return self.get(**kwargs), False
            except self.model.DoesNotExist:
                pass
            raise
```

## When NOT to use it

The solution seems like such an elegant and pragmatic one, so why wouldn't we want to use it?
Well here are two reasons:

- `get_or_create` can still fail -> There is a narrow window where the built in solution might fail, and that is under
  extreme contention, but if you are at this point, you probably know the best way to deal with it at your scale.

- Complicated creation logic -> `get_or_create` works great when the creation is self contained to a single model, but
  fails to be the correct tool once multiple models are involved. In another celery task, I had the same concurrency bug
  as this one but it was a complicated case that I needed to re-implement the `get_or_create` method just for that
  model.


Safe to say that for most applications, these two probably won't apply. If anything, probably #2 (complicated creation
logic) is the more likely reason this wouldn't work but I've only seen that every now and then. Now, back to work.
