---
layout: post
title: "Edgecases for timezones"
date: 2019-12-08
category: technical
---

Even if we use a great timezone library like `tzinfo`, a bug might still happen. That's exactly what happened to me.

I was building a graph whose x axis is of day frequency. Our database stored data in UTC. Since I wanted the days to be in accordance with local timezone, what I did was iterating through days' boundaries in local time and converting each boundary to UTC time. For example:

```
2019-03-09 00:00:00 America/Los_Angeles -> 2019-03-09 08:00:00 UTC
2019-03-10 00:00:00 America/Los_Angeles -> 2019-03-10 08:00:00 UTC
2019-03-11 00:00:00 America/Los_Angeles -> 2019-03-11 07:00:00 UTC
```

Then, I use the days' boundaries in UTC to query our database.

That code was simple. Using a timezone library helped immensely. `2019-03-10` has only 23 hours, which is correct. Beautiful.

Or so I thought.

This code was deployed, and, what a surprise, I got the error: `2019-09-08 is an invalid time`. -- Huh, what do you mean the time is invalid?

After digging in, I've found that `2019-09-08 00:00:00` doesn't exist in Chile because Chile turns the clock forward 1 hour on `2019-09-08 00:00:00` according to their daylight saving rules; `2019-09-08` starts on `01:00:00`.

We need to be aware that, (1) when we turn the clock forward, the points of time in between don't exist, and (2) when we turn the clock backward, the points of time in between occur twice. In `tzinfo` (Ruby's timezone library), converting a non-existing local time UTC raises the `PeriodNotFound` exception, and converting a twice-occurring local time to UTC raises the `AmbiguousTime` exception.

After this bug, I've researched a bit more and have found some unique timezones that are great to be included in tests:

1. __America/Santiago__ is unique because of when its daylight saving adjustments occur. On `2019-09-08`, the time jumps forward one hour when it reaches `2019-09-08 00:00:00`. So, the day actually starts on `2019-09-08 01:00:00`, and the time `[2019-09-08 00:00:00, 2019-09-08 01:00:00)` doesn't exist. My assumption that a day starts at midnight is simply false.

2. __Antarctica/Troll__ adjusts their daylight saving by 2 hours. Though the adjustment doesn't happen on a day's boundaries, it breaks my assumption that the daylight saving adjustment only happens by 1 hour.

3. __Australia/Lord_Howe__ adjusts their daylight saving by 30 minutes between UTC+10:30 and UTC+11. It breaks my assumption that the daylight saving adjustment only happens in the hour resolution.

4. __Australia/Adelaide__ is a 30-minutes timezone, and their daylight saving adjustment switches the timezone between UTC+10:30 and UTC+9:30.

5. __Australia/Eucla__ is UTC+8:45, a 45-minute timezone, that doesn't observe daylight saving.

6. __Pacific/Samoa__ changed their timezones across the international dateline, so their 30 Dec 2011 doesn't exist.

One small thing to note is, if you are designing a time-series database, using 15-minute bucket is necessary. As far as I know, all the timezones fit into the 15-minute bucket.

[IANA timezone database](https://www.iana.org/time-zones) is the official timezone database that tracks the daylight saving dates, and sometimes we forget to update our copy of the IANA timezone database. For example, for America/Santiago, [timezoneconverter.com](http://www.timezoneconverter.com/cgi-bin/zoneinfo?tz=America/Santiago) suggests that the daylight saving ends on 26 April 2020, but [zeitverschiebung.net](https://www.zeitverschiebung.net/en/timezone/america--santiago) suggests 4 April 2020. I was frustratedly stuck with writing tests on the wrong date. So, be aware of this inconsistency.

Another thing to remember is that the daylight saving's starting and ending dates are somewhat arbitrary. The dates are set by government and can be changed for varieties of reasons. For example, on 4 Oct 2018, Brazil's government announced that they would move the daylight saving's start date from 4 Nov 2018 to 18 Nov 2018 to accommodate a nation-wide university exam. Two weeks later, [Brazil's government reverted that change](https://www.timeanddate.com/news/time/brazil-postpones-dst-2018.html). So, it's important to update our copy of IANA timezone database frequently.

The lesson here is that, even if you use a timezone library, you still need to test against these unique timezones to make sure your code work correctly.

I'm sure I'm still missing a bunch of odd cases around timezones. What's your horror story on working with timezones? What interesting timezones should we add to our tests? Share your stories on [Reddit](https://www.reddit.com/r/programming/comments/e85wo4/tests_for_timezones/).
