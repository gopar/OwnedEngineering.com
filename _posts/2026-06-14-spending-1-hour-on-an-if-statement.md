---
title: Getting promoted over an if statement
date: 2026-06-14
categories: [engineering]
tags: [ownership, refactoring]
description: Small acts of ownership compound faster than you think.
---

In my earlier career, I was tasked with updating some logic in the checkout flow. When I started reading and figuring
out what part I needed to update, I was surprised to find that there was an `if` statement that was spanning 10 to 15
lines long. And this bit of code was a critical part in the whole checkout flow, and completely unreadable. I spent an
hour fixing code nobody asked me to fix. A few weeks later, that decision got me promoted.

## Unreadable code

Take a look at the following:

```javascript
if (
  (!user.hasOrderedBefore && (provider === 'new' || provider === 'ftrial')
  && !discountCode || region !== 'restricted' &&
  (discountCode && discountCode.type !== 'stacked' && !user.appliedDiscounts.length) ||
  (abTestGroup === 'B' && !user.hasOrderedBefore && provider !== 'legacy') ||
  (user.hasOrderedBefore && discountCode && discountCode.firstTimeOnly === false))
) {
  applyDiscount(...)
}
```

This bit is completely gnarly and this is some pseudo code that mimics how the `if` statement was. Sadly (or maybe
luckily) I don't remember the exact details of that particular conditional. I do however remember that it spanned around
10 lines with each line being as bad if not worse than what's shown above.

## Taking initiative

Technically to complete the task, I could just tack on another line or however many more I needed to complete the task.
Honestly, no one was going to complain if I did this because another engineer on the same team did just that a few days
before I got this ticket (I took a look at `git blame` and noticed that this person simply added more.)

The worst part is that this part of the codebase had minimal tests, so making changes could lead to regressions and
undesired behavior. So I took it upon myself to extend the test coverage before I touched anything.

Situations like this are where engineers stand out. If you want to make meaningful impact, start thinking with an
ownership mindset wherever you can. And this bit of code is a good example of how to do so. Nobody is taking the time to
make it better, no one is caring about the maintainability and extensibility of this piece of software, so now we have
the chance to shine.

Let's see how we can achieve this:


```javascript
const isFirstTimeUser = !user.hasOrderedBefore;
const isNewProvider = provider === 'new' || provider === 'trial';
const hasDiscountCode = Boolean(discountCode);
const alreadyHasDiscount = user.appliedDiscounts.length > 0;
const isInTestGroupB = abTestGroup === 'B';
const isRestrictedRegion = region === 'restricted';

const qualifiesForNewProviderDiscount = isFirstTimeUser && isNewProvider && !hasDiscountCode;
const qualifiesForStackableCode = hasDiscountCode && discountCode.type !== 'stacked' && !alreadyHasDiscount;
const qualifiesForABTestDiscount = isInTestGroupB && isFirstTimeUser && provider !== 'legacy';
const qualifiesForReturningUserCode = user.hasOrderedBefore && hasDiscountCode && !discountCode.firstTimeOnly;

const eligibleForAnyDiscount =
  qualifiesForNewProviderDiscount ||
  qualifiesForStackableCode ||
  qualifiesForABTestDiscount ||
  qualifiesForReturningUserCode;

if (eligibleForAnyDiscount && !isRestrictedRegion) {
  applyDiscount(...)
}
```

This is a drastic difference now. I made something that no one was taking the time to maintain, and turned it into
something where the next person who has to work on this part, won't spend an eternity deciphering what is going on. Now
remember the original example was pseudo code, and so the fix here is also a hypothetical but it demonstrates the
ownership idea just fine.

Surprisingly, a few weeks after I pushed out these changes, I was recommended to start on a new team that would be
making a direct impact on the business. They told me they picked me because I was showing promise of a good engineer and
cited this exact work as an example of that.

## Takeaway

There will be many chances to either pass or own pieces of work like this. Now the scope of that work may vary,
sometimes it might just be scoped to a couple of lines and sometimes its designing systems that will scale properly. But
regardless, if you don't start owning the small things and building a habit, it'll take you longer to get to those
points. I didn't realize this until much later in my career, and once I did, I started making progress significantly
faster than others around me.
