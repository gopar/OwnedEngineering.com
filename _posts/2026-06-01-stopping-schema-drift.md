---
title: Stopping Schema Drift - Part 1
date: 2026-06-01
categories: [django]
tags: []
description: Making sure a Django model mirror doesn't deviate far from the source of truth
---

We have a Django project where we store our core Django models. These Django models are used across various projects but
the issue that we are having is syncing all the models to what the source of truth repo has.


## The current solution

The way this was solved was by creating a separate repo that will hold all the Django models but with read-only
attributes and methods. That way other projects can simply install that project via Git and pip and be on their way. But
there were problems, such as some models were drifting away from what was on the source of truth repository
(new/update/dropped fields, read only fields, dependencies on abstract models, etc), engineers not updating the mirror,
and the mirror repo didn't have any way of validating code changes. So if someone commits new changes, but they forgot
to include a dependencies, wrote a field wrong, etc, then they would need to update the mirror again, tag it and
re-release it.


## Adding a test suite

Right now we'll focus on fixing the issue of validating what an engineer writes (making sure fields aren't written
incorrectly, correctly importing dependencies, etc). The way to solve this is to create a small test suite. And this
test suite should be a Django project encapsulated all by itself to verify that all the models are well written and that
there won't be any runtime problems when installing and loading this package.

Let's create the basic test suite with `pytest` as our test runner:

```bash
> tree tests
tests
├── __init__.py
├── conftest.py
├── settings.py
└── test_models.py

1 directory, 4 files
```

Now inside of `conftest.py` we'll have:

```python
import os

import django

def pytest_configure():
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "tests.settings")
    django.setup()
```

We'll have it use our own minimal Django test config:

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": ":memory:",
    }
}

INSTALLED_APPS = [
    "django.contrib.contenttypes",
    "django.contrib.auth",
    "company",
]

DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
USE_TZ = True
SECRET_KEY = "test-secret-key-not-for-production"
```

As you can see, it is extremely minimal to get a small Django project up and running.
We are only setting up the essentials, for example using an in-memory database that is SQLite, and only installing the apps that are required.

Next we have the actual tests which are the meat of the solution, split into 3 classes:

```python
"""
Tests to verify the shared models package is installable and valid.

These tests do NOT require migrations — they verify:
1. All model modules import without errors
2. Django system checks pass (catches invalid field definitions, broken relations, etc.)
3. All model classes are properly registered and introspectable
"""

import importlib
from pathlib import Path

from django.apps import apps
from django.core import checks


MODELS_DIR = Path(__file__).resolve().parent.parent / "company" / "models"


def _get_model_modules():
    """Get all Python files in company/models/ (excluding __init__ and abstract)."""
    modules = []
    for f in MODELS_DIR.glob("*.py"):
        if f.name.startswith("_") or f.name.startswith("."):
            continue
        module_name = f"company.models.{f.stem}"
        modules.append(module_name)
    return modules


class TestModelsImport:
    """Verify all model modules import without errors."""

    def test_all_model_modules_import(self):
        modules = _get_model_modules()
        assert modules, "No model modules found"

        errors = []
        for module_name in modules:
            try:
                importlib.import_module(module_name)
            except Exception as e:
                errors.append(f"{module_name}: {type(e).__name__}: {e}")

        if errors:
            raise AssertionError(
                f"Failed to import {len(errors)} module(s):\n" + "\n".join(errors)
            )

```

The first class is straight forward. We make sure we get no import errors, otherwise we'll have to come back and update
the repo again until we get it right, losing engineering velocity.

```python
class TestDjangoSystemChecks:
    """Run Django's system check framework to validate model definitions."""

    def test_no_critical_errors(self):
        """Ensure no ERROR or CRITICAL level issues from Django checks."""
        all_issues = checks.run_checks()
        errors = [
            issue
            for issue in all_issues
            if issue.level >= checks.ERROR
        ]

        if errors:
            messages = [f"{e.id}: {e.msg} (hint: {e.hint})" for e in errors]
            raise AssertionError(
                f"Django system checks found {len(errors)} error(s):\n"
                + "\n".join(messages)
            )

    def test_no_warnings(self):
        """Report warnings (non-fatal but worth knowing about)."""
        all_issues = checks.run_checks()
        warnings = [
            issue
            for issue in all_issues
            if issue.level == checks.WARNING
        ]

        if warnings:
            messages = [f"{w.id}: {w.msg}" for w in warnings]
            # Use a soft warning rather than failing
            import pytest
            pytest.skip(
                f"Django system checks produced {len(warnings)} warning(s):\n"
                + "\n".join(messages)
            )

```

Second class takes advantage of Django's `checks` module. We leverage it here to make sure the project will load
correctly, and if it doesn't we'll catch it pretty fast in CI. Please note that `test_no_warnings` prints out warnings
instead of raising them as errors. This may or may not be what you want.


```python
class TestModelIntrospection:
    """Verify models are registered and have accessible field metadata."""

    def test_all_models_have_meta(self):
        """Every registered model should have accessible _meta.get_fields()."""
        all_models = apps.get_app_config("company").get_models()
        models_list = list(all_models)

        assert models_list, "No models registered under the 'company' app"

        for model in models_list:
            fields = model._meta.get_fields()
            assert fields, f"{model.__name__} has no fields"
```

Finally the third class checks that each model has fields, otherwise we have some models that are doing absolutely
nothing. Probably an incomplete copy of what's in the source of truth repo.

## Long term fix
Now the solution that we came up with works but only solves half the problem. The problem of making sure the mirror is
synced with the source of truth is the issue that we haven't tackled yet. I will write about that part in a later post,
but for now I think this serves as an example of setting up a project like this for success. If you see a project that
needs some TLC and could benefit from something similar to this, then this will serve as an idea/guide as how to get a
fix out.
