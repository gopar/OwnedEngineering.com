---
title: Letting the user fix stale data without knowing it
date: 2026-07-01
categories: [engineering]
tags: [ownership]
description:
---

In our application, we would allow users to save their preference and then load those preferences when they returned.
The problem arose when those preferences no longer existed, this would cause our application to be in a weird state,
since we didn't account for this scenario, but found a solution.

## Backstory

When a user would view details of a specific entity in our database, we would show related data as well. The thing is
that some of that related data was being fetched from a third party and it involved multiple steps for the user to view
that data. Hence we would save that preference (flow) that the user chose, so they don't have to do it every time.

As some of you might insinuate, the problem starts with it being fetched from a third party, so sometimes the data that
we saved no longer existed. When this happened our frontend would be partially broken. Once this came to our attention,
a ticket was written to propose a fix.

The fix involved creating a cron job that would go through customers preferences and see if they still existed by
querying the third party API. If they didn't, then delete that preference, and our frontend could behave as expected.

Here is a rough flow of our frontend logic:


```
Page is loaded
|
Fetch preference
|
Do some work with it
|
Fetch all possible data
|
Check preference exists in data
|
we call third party library to load preference or start initial flow if none
|
If initial flow, user picks new preference and we store that
```

The part where it would break was where it would try to do work with the fetch preference.

## The solution

I ended up picking up the ticket and at first I agreed with the solution not thinking much of it but halfway through I
realized something. Our database was already being heavily used, so I didn't like the idea of adding more tasks to it,
even if we were running the cron jobs on lower peak times. I also didn't like querying another third party service that
would go against our API limits (not to mention it would sometimes take a few seconds to get a respond back so it wasn't
the fastest)

The fix was pretty simple. When we try to read the users preference and the saved data that we have is stale, then we'll
pretend that there is no saved data. With this in mind, the frontend will act as if its the first time, and allow the
user to choose their preference.

This means that we switched the order of things:

```
Page is loaded
|
Fetch preference
|
Fetch all possible data
|
Check if preference exists in fetched data
|
If found, use it. If not, treat as first-time flow.
|
we call third party library to load preference or start initial flow if none
|
If initial flow, user picks new preference and we store that
```

Now our database isn't taking any extra load, and we aren't creating any additional queries to the third party API.

## Takeaway

It's our job to think about these problems and try to solve them in the optimal/best way possible given our constraints.
The proposed solution makes sense since it does solve the problem and it's a common pattern that many do, but so is the
solution I ended up implementing.

All this to say to spend a few minutes to figure out if you can do better, think of it as a mini challenge or exercise
to always improve.
