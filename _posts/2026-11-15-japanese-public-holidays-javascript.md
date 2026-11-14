---
layout: post
title: "Japanese public holidays in 60 lines of JavaScript (substitute holidays, equinoxes and the \"sandwich\" rule)"
description: "All Japanese public holiday rules (fixed dates, Happy Monday, equinoxes, substitute and citizens' holidays) with a tested dependency-free JavaScript implementation."
tags: [javascript, japan, datetime, tutorial]
---

If your app shows delivery dates, due dates or opening hours in Japan, you need Japanese public holidays. The usual approach is a hard-coded list, which works until someone forgets to update it in December.

The rules are actually short. Here is the full set that has been in force since 2022, and a dependency-free implementation you can paste into a project.

## The rules

**Fixed dates** (10): Jan 1, Feb 11, Feb 23, Apr 29, May 3, May 4, May 5, Aug 11, Nov 3, Nov 23.

**"Happy Monday" holidays** (4) fall on the nth Monday of a month:

| Holiday | Rule |
|---|---|
| Coming of Age Day | 2nd Monday of January |
| Marine Day | 3rd Monday of July |
| Respect for the Aged Day | 3rd Monday of September |
| Sports Day | 2nd Monday of October |

**Equinox days** (2) are officially announced every February for the next year, based on astronomical observations. In practice a well-known approximation matches the announced dates for 1980–2099:

```
vernal   = floor(20.8431 + 0.242194 * (y - 1980) - floor((y - 1980) / 4))   // March
autumnal = floor(23.2488 + 0.242194 * (y - 1980) - floor((y - 1980) / 4))   // September
```

**Substitute holiday (振替休日):** when a holiday falls on a Sunday, the next day that is *not already a holiday* becomes a holiday. That "not already a holiday" part matters in Golden Week: if May 3 is a Sunday, the substitute day is May 6, not May 4.

**Citizens' holiday (国民の休日):** an ordinary day sandwiched between two holidays becomes a holiday. Today this can only happen in September, when Respect for the Aged Day and the Autumnal Equinox are two days apart (2026 is one of those years: Sep 21, **22**, 23).

## The code

```js
const nthMonday = (y, m, n) => {
  const first = new Date(Date.UTC(y, m - 1, 1)).getUTCDay(); // 0 = Sunday
  return 1 + ((8 - first) % 7) + (n - 1) * 7;
};
const vernal = (y) => Math.floor(20.8431 + 0.242194 * (y - 1980) - Math.floor((y - 1980) / 4));
const autumnal = (y) => Math.floor(23.2488 + 0.242194 * (y - 1980) - Math.floor((y - 1980) / 4));
const iso = (y, m, d) => new Date(Date.UTC(y, m - 1, d)).toISOString().slice(0, 10);

export function holidays(y) {
  const h = new Map([
    [iso(y, 1, 1), "New Year's Day"],
    [iso(y, 1, nthMonday(y, 1, 2)), 'Coming of Age Day'],
    [iso(y, 2, 11), 'National Foundation Day'],
    [iso(y, 2, 23), "Emperor's Birthday"],
    [iso(y, 3, vernal(y)), 'Vernal Equinox Day'],
    [iso(y, 4, 29), 'Showa Day'],
    [iso(y, 5, 3), 'Constitution Memorial Day'],
    [iso(y, 5, 4), 'Greenery Day'],
    [iso(y, 5, 5), "Children's Day"],
    [iso(y, 7, nthMonday(y, 7, 3)), 'Marine Day'],
    [iso(y, 8, 11), 'Mountain Day'],
    [iso(y, 9, nthMonday(y, 9, 3)), 'Respect for the Aged Day'],
    [iso(y, 9, autumnal(y)), 'Autumnal Equinox Day'],
    [iso(y, 10, nthMonday(y, 10, 2)), 'Sports Day'],
    [iso(y, 11, 3), 'Culture Day'],
    [iso(y, 11, 23), 'Labour Thanksgiving Day'],
  ]);
  const day = (s) => new Date(s + 'T00:00:00Z');
  const add = (s, n) => { const d = day(s); d.setUTCDate(d.getUTCDate() + n); return d.toISOString().slice(0, 10); };

  // Citizens' holiday: an ordinary day sandwiched between two holidays.
  for (const s of [...h.keys()]) {
    const mid = add(s, 1);
    if (h.has(add(s, 2)) && !h.has(mid) && day(mid).getUTCDay() !== 0) h.set(mid, "Citizens' Holiday");
  }
  // Substitute holiday: a holiday on Sunday moves to the next non-holiday.
  for (const s of [...h.keys()].sort()) {
    if (day(s).getUTCDay() !== 0) continue;
    let t = add(s, 1);
    while (h.has(t)) t = add(t, 1);
    h.set(t, 'Substitute Holiday');
  }
  return [...h].sort(([a], [b]) => a.localeCompare(b)).map(([date, name]) => ({ date, name }));
}
```

Two details that bite people:

1. **Use UTC dates.** `new Date(2026, 4, 3)` is local time; on a server in UTC−8 it can silently become May 2. Build dates with `Date.UTC` and compare ISO strings.
2. **Apply the sandwich rule before the substitute rule**, and when moving a substitute day, skip days that are already holidays.

## Business days

Most real use cases are "N business days from today", skipping weekends and holidays:

```js
export function addBusinessDays(start, n) {
  const d = new Date(start + 'T00:00:00Z');
  const cache = {};
  const isHoliday = (s) =>
    (cache[s.slice(0, 4)] ||= new Set(holidays(+s.slice(0, 4)).map((x) => x.date))).has(s);
  while (n > 0) {
    d.setUTCDate(d.getUTCDate() + 1);
    const s = d.toISOString().slice(0, 10), wd = d.getUTCDay();
    if (wd !== 0 && wd !== 6 && !isHoliday(s)) n--;
  }
  return d.toISOString().slice(0, 10);
}

addBusinessDays('2026-04-28', 5); // '2026-05-11'
// Skips Apr 29, the weekend and May 4-6. May 3, 2026 is a Sunday,
// so its substitute holiday lands on May 6 (May 4 and 5 are already holidays).
```

## How I checked it

I compared the output for every year from 2022 to 2050 against the holiday table I maintain for an API, which itself is checked against the Cabinet Office's published holiday CSV (1955–2027). Every date matched.

## Limits

- The law can change. It did in 2019–2021 (the new Emperor's birthday, and holidays moved for the Olympics), so years before 2022 need extra cases.
- Equinox days beyond the officially announced year are a forecast, not a guarantee.
- One-off holidays (like the 2019 enthronement) are not covered by any formula.

## If you'd rather not maintain this

I also run a small hosted API, **Japanese Text Toolkit**, on RapidAPI. It returns holidays with Japanese and English names, counts and adds business days, converts dates to the Japanese era (Reiwa 8), and converts kana to romaji. There is a free plan with 500 requests a month:

- RapidAPI: https://rapidapi.com/sutorimasutorima/api/japanese-text-toolkit
- Examples (Node.js and Python): https://github.com/sutorimasomurie-afk/japanese-text-toolkit-api

The code in this post is free to copy, no API needed.

---

*This post was drafted with help from an AI assistant. The code was run and checked against real data before publishing.*
