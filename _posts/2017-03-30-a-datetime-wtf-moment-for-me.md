---
layout: post
title: A DateTime WTF moment for me
---

I am working on a small webapp to manage my BJJ academy and today I noticed something weird on my dashboard. I have a small widget that displays the number of signups for each of the last 3 months like below.

![The bug]({{ site.url }}/public/datetime-wtf-bug.png)

So what happened?

To get the last 3 months, I wrote a very simple method:

```php
private function getLastMonths(int $monthCount): array
{
    $months = [];
    for ($i = 0; $i < $monthCount; $i++) {
        $date = new DateTimeImmutable("-{$i} month");
        $months[] = new Month($date->format('m'), $date->format('Y'));

    }

    return $months;
}
```

Can you spot the bug?

The problem was that today is March 30th. The '-1 month' tries to set the date to February 30th which of course doesn't exist, so instead it ended up as March 2nd.

It was a very easy fix once I figured out what the problem was.

```php
private function getLastMonths(int $monthCount): array
{
    $firstOfTheMonth = new DateTimeImmutable('first day of this month');

    $months = [];
    for ($i = 0; $i < $monthCount; $i++) {
        $date = $firstOfTheMonth->modify("-{$i} month");
        $months[] = new Month($date->format('m'), $date->format('Y'));
    }

    return $months;
}
```

Everything is back to normal.

![Everything back to normal]({{ site.url }}/public/datetime-wtf-fixed.png)

This really caught me off guard even though I've been using PHP for the last 12 years I never realized this. I was lucky to learn this lesson with something so insignificant, but now I'm left wondering if someone somehwere had a bug reported today because of some old code of mine...
