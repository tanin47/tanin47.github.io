---
layout: post
title: "Use SQL instead of Spreadsheet/Excel"
description: "SQL can handle large dataset, is more ergonomic, and is faster. If you work with CSVs, you should consider using SQL."
date: 2021-10-02
category: main
---

Since starting at a new company, I've been working on an accounting report that comes in the form of CSVs, so I've worked with a lot of CSVs.

As a person who knows SQL well but isn't good at Excel, working with CSV files in Excel is such a nightmare. 
Every time I was doing something, I felt that I would have spent 10x less time if I could use SQL.

VLOOKUP and pivot tables are absolutely difficult to use. It's not just writing stuff. It's a combination of coding and meddling with UI.

SQL is much more ergonomic. It's just a piece of text. There are few benefits of being "a piece of text":

1. You can save the SQL in a file and reuse it on different CSV files that have the same format.
2. It's easier to check for correctness because the whole logic is contained within a single SQL.

Besides the above, there's a blocker where Excel cannot load a CSV with more than 1M rows. Using SQL with an actual database won't encounter this blocker.

In this blog, I'd like to show how VLOOKUP and pivot tables can be replaced with SQL, and hopefully this would inspire you to learn the basic of SQLs.

### VLOOKUP

VLOOKUP is somewhat equivalent to `JOIN` in SQL. 

Let's say there are 2 CSV files that are associated to each other through `employee_id`.

|employee_id|name|
|-----------|----|
|1|John|
|2|Jane|
|3|Mary|

and

|employee_id|salary|
|-----------|------|
|1|70000|
|2|80000|
|3|60000|

We can write a SQL that will construct a table that contains `name` and `salary` on the same row:

```
SELECT 
  n.employee_id, 
  n.name,  
  s.salary 
FROM names n 
JOIN salaries s 
ON n.employee_id = s.employee_id
```

The above SQL yields:

|employee_id|name|salary|
|-----------|----|------|
|1|John|70000|
|2|Jane|80000|
|3|Mary|60000|

You can see that it's just a short SQL to achieve what VLOOKUP can do.

### Pivot tables

Pivot tables is somewhat equivalent to `GROUP BY`. 

Let's say there's one CSV that contains sales per location, and you want to get total sales per location. Here's the CSV:

|state|price|
|-----|-----|
|CA|10|
|CA|12|
|WA|17|
|WA|1|
|NY|9|

We can write a SQL that will construct a table that group by `state` and compute the total sales of each state:

```
SELECT 
  state, 
  SUM(price) as price 
FROM sales 
GROUP BY state
```

The above SQL yields:

|state|price|
|-----|-----|
|CA|22|
|WA|18|
|NY|9|

If you are familiar with SQL, this would take less than a minute to write.

### What next?


Since you read up until this point, I know the question that you are having now is: "ok, I'm convinced. But where can I write SQL on CSVs?".

I had asked the very same question, and there are many options like:

* [Microsoft Access](https://www.microsoft.com/en-us/microsoft-365/access)
* [Q](http://harelba.github.io/q/) (Run SQL directly on CSV on command-line tool)
* [SQLite command-line tool](https://www.sqlite.org/cli.html) which supports importing CSV.

However, I ended up building my own Desktop application for this purpose: [Superintendent.app](https://superintendent.app), which supports Windows, Mac, and Linux.

I intend for Superintendent.app to be an SQL-focused spreadsheet Desktop app that offers great UX like Excel does (e.g. sort columns, sum values on the selected cells).
It's not there today, but we are moving toward that directly.

Since Superintendent.app is backed by [SQLite](https://www.sqlite.org), it is very fast (e.g. loading 1GB file within 10s on Macbook Pro 2020) and can handle a dataset as large as your memory allows.

While I want you to try Superintendent.app, the main gist is to show you how SQL can significantly improve your workflow, and hopefully you are inspired to start learning SQL.

Thank you for reading until the end, and I'll see you next time :)
