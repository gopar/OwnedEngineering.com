---
title: Hidden Cost of Django Rest Framework Pagination
date: 2026-05-22
categories: [django]
tags: [Django Rest Framework]
description: Django Rest Framework, just like Django, always tries to be helpful, but it might not always be beneficial.
---

Django pagination has 2 problems. Both of which will start to hurt as you scale.

We'll look at these problems through the eyes of Django Rest Framework (DRF) since it also uses Django's pagination internally. Specifically through:

- CursorPagination
- LimitOffsetPagination
- PageNumberPagination

## Table of Contents

Note: All of code sources are from Django 6.0 and DRF 3.16.1

## A look at LimitOffsetPagination

Let's start by looking at the source code of `LimitOffsetPagination` (rest_framework/pagination.py):

```python
class LimitOffsetPagination(BasePagination):
    def paginate_queryset(self, queryset, request, view=None):
        #...
        self.count = self.get_count(queryset)
        self.offset = self.get_offset(request)
        #...
        return list(queryset[self.offset:self.offset + self.limit])

    def get_paginated_response(self, data):
        return Response({
            'count': self.count,
            'next': self.get_next_link(),
            'previous': self.get_previous_link(),
            'results': data
        })

    def get_count(self, queryset):
        try:
            return queryset.count()
        except (AttributeError, TypeError):
            return len(queryset)
```

I've truncated the code to show only relevant parts.

`LimitOffsetPagination` gets it's name from the SQL method it uses to paginate data (via `OFFSET`). It's been well
documented that `OFFSET` is not ideal when there is a large amount of data, because it ends up reading all the data from
the beginning of the query up to where it truncates. For example, if we want to fetch the next 50 entries after 1,000th
place, then the database has to read all 1,000 entries plus the next 50 to return, making it really wasteful once we
reach a heavily populated table.

This alone is rather wasteful and would deter some engineers from even adding this as a pagination option, but that is
not the only issue with this type of pagination (specifically the class implementation).

If we look at what's return from `get_paginated_response`, we can see that the payload has a key of `count`. Now if we
trace where it originates (`get_count`), we then realize another unpleasant truth. And that is, that Django will do a
`.count()` call on the queryset, which means that a sequential scan will have to happen in order to get an accurate
count of rows in the table. On a large table, this might cross the acceptable threshold of a response time, plus it adds
a second query that needs to be executed **every** time we run through this API endpoint.

## A look at PageNumberPagination

If we look at `PageNumberPagination`, we'll see that it's not much of an improvement:

```python
class PageNumberPagination(BasePagination):
    def get_paginated_response(self, data):
        return Response({
            'count': self.page.paginator.count,
            'next': self.get_next_link(),
            'previous': self.get_previous_link(),
            'results': data,
        })

    def get_next_link(self):
        if not self.page.has_next():
            return None
        url = self.request.build_absolute_uri()
        page_number = self.page.next_page_number()
        return replace_query_param(url, self.page_query_param, page_number)

    def get_previous_link(self):
        if not self.page.has_previous():
            return None
        url = self.request.build_absolute_uri()
        page_number = self.page.previous_page_number()
        if page_number == 1:
            return remove_query_param(url, self.page_query_param)
        return replace_query_param(url, self.page_query_param, page_number)
```

Under the hood `PageNumberPagination` uses the `Paginator` class from `django.core.paginator` which has the same issues.
When calling `self.page.paginator.count`, its doing that same sequential scan and caching it (Cache only useful within
that same request). Both `get_next_link` and `get_previous_link` don't help either since they rely on
`self.page.has_next()` and `self.page.has_previous` internally. Both of which check against `self.paginator.count` to
determine if more pages exist. This means you can't skip the `COUNT(*)` query for knowing whether a next or previous
link should even be generated. No count, no links.

## A Look At CursorPagination

Let's see if `CursorPagination` is any better:

```python
class CursorPagination(BasePagination):
    def get_paginated_response(self, data):
        return Response({
            'next': self.get_next_link(),
            'previous': self.get_previous_link(),
            'results': data,
        })
 ```
Right off the bat we can see that there is no `count` attribute in the response,
meaning that this won't cost us an extra count or offset query like we've seen in the previous two examples.

This plus of removing the count/offset query doesn't come without its downsides. For one, the query needs to be
deterministic (which is a good practice to have anyways) and doesn't allow for random "page" access. This may be a deal
breaker for those who require it.

Though a perfect example of when to use this, is when you are implementing infinite scrolling.

## Always use CursorPagination?

As usual, it's not easy to say to always use x. You first need to discuss the trade-offs. For example, you can run
analytics to see if users are randomly accessing pages or if they're simply going sequentially. Clicking next, next and
previous, previous. If that's the case, then I would argue that cursor pagination is the best option there. Another
scenario where cursor pagination may be a good option is when the table size is rather big. As we mentioned, the larger
the table, the slower the count and offset queries will become. If that's not a worry for you, then perhaps the
limit/count classes are the better option since you can show the user how many objects and pages remain.
