---
layout: post
title: "Analyze log files with SQL using Superintendent.app"
description: "Superintendent.app allows you to load log files and write SQL on them. This post shows a couple basic examples how we analyze log files with SQL."
date: 2021-11-24
category: main
---

I've launched [Superintendent.app](https://superintendent.app) 6 months ago. Initially, I built it to make me more productive at work since I work with
a lot of CSV files, and I loathe using Excel. After building it, I've realized that having a Desktop app that can handle GBs of
CSV files is really versatile. I can analyze CSV files from customers, cross-check (i.e. join) them with internal data, reformat them in 
some ways, and etc.

Analyzing log files is another interesting use case. I'm sure there are tons of tools that can do the same thing, though I'm not sure
if those tools are backed by a real database. Superintendent.app is backed by Sqlite, a real database, which can result in a different
set of strengths and weaknesses e.g. medium initial load time (10s for 1GB file), quick query time, and powerful functionality.

To recap, Superintedent.app is an offline Desktop app that allows you to import CSV/TSV files (and several other tabular formats) and 
write SQLs on those files. Here's how the app looks like:

<video width="100%" controls>
    <source src="/assets/img/analyze-log/superintendent_v1_1_0.mp4" type="video/mp4" />
</video>

As a desktop app, Superintendent.app can handle GBs of files easily. Moreover, the privacy and security aspect of an offline 
Desktop app yields better peace of mind.

Superintendent.app also enables you to cross-check (i.e. join) records between the log file and a database table (exported in CSV).

Since Superintendent.app is backed by Sqlite, which is powerful but still quite limited in SQL capabilities compared to other SQL-style databases), 
we've added 3 convenient SQL functions to make our lives easier, namely, [date_parse](https://docs.superintendent.app/functions/date-parse), 
[regex_extract](https://docs.superintendent.app/functions/regex-extract), and [regex_replace](https://docs.superintendent.app/functions/regex-replace).

Because I'm a big fan of papertrail.com, I'd like to follow their example of reading log files: [https://www.papertrail.com/help/permanent-log-archives](https://www.papertrail.com/help/permanent-log-archives)

Instead of using command-line, I'll use Superintendent.app to perform the same task.

### Setup

I'll use the log files from [https://superintendent.app](https://superintendent.app) as an example. The schema of a log file looks like below:

```
id
generated_at
received_at
source_id
source_name
source_ip
facility_name
severity_name
program
message
```

We'll need to add the header row as the first row manually since a log file doesn't have one.

After adding the header row, the file looks like below:

```
id  generated_at  received_at source_id source_name source_ip facility_name severity_name program message
1400180097685377024 2021-11-24 15:02:51 2021-11-24 15:02:51 8113625781  superintendent-prod 3.310.241.96  Local0  Error app/postgres.1262692  [DATABASE] [9-1]  sql_error_code = 28000 FATAL:  no pg_hba.conf entry for host "48.136.164.110", user "postgres", database "postgres", SSL off
1400180234126229518 2021-11-24 14:59:22 2021-11-24 15:03:24 8113625781  superintendent-prod 10.215.223.49 Local0  Info  app/heroku-postgres source=DATABASE addon=postgresql-spherical-57847 sample#current_transaction=1879 sample#db_size=10678831bytes sample#tables=3 sample#active-connections=22 sample#waiting-connections=0 sample#index-cache-hit-rate=0.99313 sample#table-cache-hit-rate=0.98474 sample#load-avg-1m=0 sample#load-avg-5m=0 sample#load-avg-15m=0 sample#read-iops=0 sample#write-iops=0.074205 sample#tmp-disk-used=542765056 sample#tmp-disk-available=72436027392 sample#memory-total=4023192kB sample#memory-free=178744kB sample#memory-cached=3279752kB sample#memory-postgres=53508kB sample#wal-percentage-used=0.06477739851760235
1400181318660173834 2021-11-24 15:06:27 2021-11-24 15:07:42 8113625781  superintendent-prod 56.92.230.219 Local0  Info  app/heroku-postgres source=DATABASE addon=postgresql-spherical-57847 sample#current_transaction=1879 sample#db_size=10678831bytes sample#tables=3 sample#active-connections=22 sample#waiting-connections=0 sample#index-cache-hit-rate=0.99313 sample#table-cache-hit-rate=0.98474 sample#load-avg-1m=0 sample#load-avg-5m=0 sample#load-avg-15m=0 sample#read-iops=0 sample#write-iops=0.097046 sample#tmp-disk-used=542765056 sample#tmp-disk-available=72436027392 sample#memory-total=4023192kB sample#memory-free=177796kB sample#memory-cached=3279804kB sample#memory-postgres=53508kB sample#wal-percentage-used=0.06477739851760235
...more rows...
```

Loading the TSV into Superintendent.app yields the screenshot below:

![Initial load](/assets/img/analyze-log/analyze-log-first-load.jpg)

Now we can follow the usage examples explained here: [https://www.papertrail.com/help/permanent-log-archives](https://www.papertrail.com/help/permanent-log-archives)

### Usage example

__Show identical messages__

Papertrail suggests the command below:

```
gzip -cd 2021-11-24-15.tsv.gz | cut -f10 | sort | uniq -c | sort -n
```

An equivalent SQL is:

```
SELECT COUNT(*), message FROM "2021_11_24_15" GROUP BY message ORDER BY COUNT(*) DESC
```

It would yield a result as shown below:

![Identify identical messages](/assets/img/analyze-log/analyze-log-identical-message.jpg)

__Show similar message__

Papertrail suggests the command below:

```
$ gzip -cd *.tsv.gz  | # extract all archives
    cut -f 5,9-      | # sender, program, message
    tr -s '\t' ' '   | # squeeze whitespace
    tr -s 0-9 0      | # squeeze digits
    cut -d' ' -f 1-8 | # truncate after eight words
    sort | uniq -c | sort -n
```

An equivalent SQL is:

```
WITH sanitized AS (
  SELECT
    REGEX_EXTRACT(
      '^(([^\s]+ ){0,8})', -- Get the first 8 words
      REGEX_REPLACE('[0-9]+', REGEX_REPLACE('\s+', message, ' ', false), '0', false) -- Squeeze whitespaces and digits
    ) AS message,
    REGEX_REPLACE('[0-9]+', REGEX_REPLACE('\s+', program, ' ', false), '0', false) AS program,
    REGEX_REPLACE('[0-9]+', REGEX_REPLACE('\s+', source_name, ' ', false), '0', false) AS source_name
  FROM "2021_11_24_15"
)

SELECT COUNT(*) AS occurence, message FROM sanitized GROUP BY message ORDER BY COUNT(*) DESC
```

The above SQL yields the result below:

![Identify similar messages](/assets/img/analyze-log/analyze-log-similar-message.jpg)

I wish Sqlite supports define a user-defined function within a SQL statement,
so the code will be cleaner. But this is an improvement for another day.

__Searching__

Searching is the simplest one because you can use `column LIKE '%something%'` or `regex_extract('regex', column) IS NOT NULL` in order to search for a specific string.

For example, if you want to search for the log lines where `write-iops` is more than or equal to `0.087`, you can use the SQL below:

```
SELECT
  CAST(regex_extract('write-iops=([0-9\.]+)', message) AS decmial) AS write_iops,
  regex_extract('(.{0,30}write-iops=[0-9\.]+.{0,30})', message) AS context, -- Show context
  *
FROM "2021_11_24_15"
WHERE write_iops >= 0.087
```

The SQL yields the result as shown below:

![Search](/assets/img/analyze-log/analyze-log-seach.jpg)

Beautiful, isn't it?

### Wrapping up

If you are familiar with SQL, using SQL to analyze log files makes a lot of sense. You can visit [https://superintendent.app](https://superintendent.app) to try it out.

Thank you for reading until the end. I'll see you next time.
