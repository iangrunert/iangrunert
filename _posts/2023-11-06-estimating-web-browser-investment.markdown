---
layout: post
title:  "Estimating Web Browser Investment"
---

On Brian Kardell's post [Where Browsers Come From](https://bkardell.com/blog/WhereBrowsersComeFrom.html) there's an
estimate given on the cost of maintaining the three major browser engines:

> What does the browser commons cost, per-user, per-year?

> Based on what we know of team sizes, scales and budgets - several people seem to have arrived on a similar ballpark
> figure: Somewhere around 2 billion dollars per year.

This figure feels like an overestimate to me, so let's dig a little further into this. I want to get my own very rough
estimate to see if I reach a similar ballpark figure.

A reasonable place to start is the commit logs - while it won't capture everyone who contributes to the browser
commons, it'll give us a lower bound from which we can estimate the number of additional roles (managers, developer
relations, tech writers, etc).

We'll remove some duplicates, and free gmail addresses (as a likely signal they, like me, are not being paid). 
Otherwise we'll be very conservative in saying that all committers to the three major engines are employed
full-time to advance the browser commons. That's definitely not true, but I'm trying to find a more reasonable upper
bound.

I'll be looking at commits from the 2022 calendar year. While it was tempting to look at the trailing 12 months, having
a repeatable time frame is useful if people disagree with my analysis. We'll collate the email addresses from the three
engines into one file for de-duplication. This will prevent double counting some people from Igalia who may contribute
to multiple engines in the same calendar year.

Then, with the number of committers we can determine a rough estimate of their total compensation. After that we'll
add 50% for a conservative upper-bound of the cost of management, developer relations, tech writers, bug 
reporters, build infrastructure cost, sales (to negotiate the default search deal occasionally). This won't include
cost of senior management, expenditure for hardware platforms, or expenditure to try and diversify income streams - 
just trying to get a rough estimate on maintaining and improving the browser engines themselves.

### WebKit:

We'll start with WebKit, because I'm more familiar with the project.

```
$ git log --since "Jan 1 2022" --before "Jan 1 2023" --format="%ae" | grep -v gmail.com | cut -d'@' -f1,2 | sort | uniq | wc -l
269
```

The `cut -d'@' -f1,2` removes duplicates due to trailing UUIDs in git authors:

```
a@example.com@
a@example.com@00000000-0000-0000-0000-000000000000
```

269 is a bit of an overestimate: there's 207 from Apple, Igalia and Sony. Let's roll with it for now, we're just
looking to get an upper bound.

### Firefox (Gecko):

I'm using the [gecko-dev](https://github.com/mozilla/gecko-dev) mirror.

```
$ git log --since "Jan 1 2022" --before "Jan 1 2023" --format="%ae" | grep -v gmail.com | cut -d'@' -f1,2 | sort | uniq | wc -l
839
```

That's a surprise to me! I had expected Gecko to be in the same ballpark as WebKit. Let's look at the top 10 domains:

```
$ git log --since "Jan 1 2022" --before "Jan 1 2023" --format="%ae" | grep -v gmail.com | cut -d'@' -f1,2 | sort | uniq | cut -d'@' -f2 | sort | uniq -c | sort -n --reverse | head -n 10
    244 mozilla.com
    169 chromium.org
     55 google.com
     42 users.noreply.github.com
     27 igalia.com
     22 apple.com
     14 microsoft.com
      9 outlook.com
      7 intel.com
      6 protonmail.com
```

Ah, okay. The Chromium, Google, Apple and (potentially) Igalia ones may end up being double-counted. There's some free
e-mails in here and some github noreply e-mail addresses which likely aren't being paid. I'll ignore those for now,
we're just looking to get an upper bound. Might revisit this later, there's a long tail of single-commit authors who
are unlikely to be getting paid.

### Chromium (Chrome):

Chrome uses git submodules so makes it a little more difficult of a task. Let's start in src:

```
$ git log --since "Jan 1 2022" --before "Jan 1 2023" --format="%ae" | grep -v gmail.com | cut -d'@' -f1,2 | sort | uniq | wc -l
2616
```

Woah! Another surprising result, that's much higher than I expected. That's a lotta contributors! Let's look at the top
10 domains:

```
$ git log --since "Jan 1 2022" --before "Jan 1 2023" --format="%ae" | grep -v gmail.com | cut -d'@' -f1,2 | sort | uniq | cut -d'@' -f2 | sort | uniq -c | sort -n --reverse | head -n 10
   1132 google.com
   1105 chromium.org
    101 microsoft.com
     41 intel.com
     33 igalia.com
     26 yandex-team.ru
     20 samsung.com
     11 opera.com
     11 navercorp.com
      9 chops-service-accounts.iam.gserviceaccount.com
```

Ok, maybe there's some duplication here between google.com and chromium.org. Committers can gain a chromium.org e-mail
address, and most use the same username as their google.com address. I'll do some spot inspection first:

```
git log --since "Jan 1 2022" --before "Jan 1 2023" --format="%ae" | grep -e chromium.org -e google.com | cut -d'@' -f1,2 | sort | uniq | less
```

Then will figure out a way to count the duplication:

```
$ git log --since "Jan 1 2022" --before "Jan 1 2023" --format="%ae" | grep -e chromium.org -e google.com | cut -d'@' -f1,2 | sort | uniq -c | wc -l
2241
$ git log --since "Jan 1 2022" --before "Jan 1 2023" --format="%ae" | grep -e chromium.org -e google.com | cut -d'@' -f1 | sort | uniq -c | wc -l
2038
```

Okay, only 203 duplicates. The Chrome team is clearly much larger than the WebKit and Firefox teams.

Let's also grab src/v8 contributors. I don't want to delve too deep into the codebase for a rough figure, but that
feels like an important component to include.

```
$ git log --since "Jan 1 2022" --before "Jan 1 2023" --format="%ae" | grep -v gmail.com | cut -d'@' -f1,2 | sort | uniq | wc -l
191
```

### Summary

Combining those e-mail lists together, we get:

```
$ # total contributors
$ cat contributors-chromium.txt contributors-gecko.txt contributors-v8.txt contributors-webkit.txt | sort | uniq | wc -l
3478
$ # all chromium.org + google.com emails
$ cat contributors-chromium.txt contributors-gecko.txt contributors-v8.txt contributors-webkit.txt | grep -e chromium.org -e google.com | cut -d'@' -f1,2 | sort | uniq -c | wc -l
2275
$ # removing duplicate usernames from chromium.org + google.com emails
$ cat contributors-chromium.txt contributors-gecko.txt contributors-v8.txt contributors-webkit.txt | grep -e chromium.org -e google.com | cut -d'@' -f1 | sort | uniq -c | wc -l
2063
$ echo "$(( 3478 - (2275 - 2063) ))"
3266
```

So eliminating 212 likely google / chromium duplicates, we're at 3266 contributors with 59% of them being Chromium /
Google. Assuming the **average** compensation is somewhere between a Google L4 and L5 at $320K, that's $1,045,120,000.
Adding 50% overhead we're at $1,567,680,000 - roughly $1.6 billion.

The 50% overhead is likely an overestimate, the average compensation might be an overestimate, and there is a long tail
of small contributions which probably don't represent full-time employment on advancing the browser commons. There's
probably better ways to de-duplicate contributors who use multiple e-mail addresses. I could've spent more time in the 
Chromium project to get a more accurate picture of the number of contributors through enumerating all the submodules.

However the 2 billion per year figure isn't as outlandish as I thought! I'd been more familiar with the WebKit project
and hadn't accounted for just how wildly different the level of investment is between WebKit and Chromium. I can see 
how you might get to $2 billion with different methodology and likely more information than I have.