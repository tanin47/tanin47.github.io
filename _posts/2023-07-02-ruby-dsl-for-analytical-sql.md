---
layout: post
title: "An open-sourced Ruby DSL for composing maintainable analytical SQL statements"
description: "The DSL supports higher order primitives like parameterization and meta-programming, which makes writing, and resuing parts of, SQL statements easier. This is suitable for an application that builds analytics on top of SQL-supported data warehouses like Presto."
date: 2023-07-02
---

Repository: [https://github.com/tanin47/lilit-sql](https://github.com/tanin47/lilit-sql)

At the company I work at, we are building an analytics product on top of Presto, which supports SQL as its interface. We have a lot of long SQL statements to power charts, tables, and CSVs. When I say long, a final SQL statement is likely >2,000 lines.

One major pain point arises. There are common parts that can be re-used by multiple SQL statements. It would be annoying to keep duplicate these common parts, so we currently solve this pain point by refactoring certain parts into Ruby's ERBs. However, this is extremely brittle because SQL is a declarative language. It requires the common part to be aware of the columns of the previous statements.

Consider the below example where there are 3 reports:

1. MRR report that has the following columns: `month`, `mrr_amount`, `customer_id`, `customer_name`
2. Payment report that has the following columns: `charge_id`, `timestamp`, `amount`, `customer_id`, `customer_name`, `invoice_id`, `invoice_number`
3. Unpaid invoice report that has the following columns: `invoice_id`, `unpaid_amount`, `age`, `invoice_number`

The common part that we can see from this report is the `customer_name` and `invoice_number` column where we can get them by joining with the `customers` and `invoices` table. You can imagine it could be implemented as below if SQL was more advanced:

```sql
select
  main.*,
  -- if main contains customer_id
    customers.name as customer_name
  -- end
  -- if main contains invoice_id
    invoices.number as invoice_number
  -- end
from
  main
  -- if main contains customer_id
    left join customers
    on main.customer_id = customers.id
  -- end
  -- if main contains invoice_id
    left join invoices
    on main.invoice_id = invoices.id
  -- end
```

The above pattern would have been fine if SQL supported the capability to check whether a certain column exists in the previous SQL statement before writing the next SQL.

But SQL doesn't support that!

This is why lilit-sql, the Ruby DSL for composing maintainable SQL statements, is built, and we can achieve the same thing with:

```ruby
Customer = Struct.new(:id, :name)
Invoice = Struct.new(:id, :number)

def with_lookup_columns(query)
  if query.has?(:customer_id)
    customers = Query.from(Table.new(Customer, 'customers'))
    query = query.left_join(customers) do |*tables|
      left = tables.first
      invoice = tables.last

      left.customer_id == customer.id
    end
  end

  if query.has?(:invoice_id)
    invoices = Query.from(Table.new(Invoice, 'invoices'))
    query = query.left_join(invoices) do |*tables|
      left = tables.first
      invoice = tables.last

      left.invoice_id == invoice.id
    end
  end
end
```

You can see the full example in [README.md](https://github.com/tanin47/lilit-sql)

lilit-sql goes even further and supports "dynamic" columns as well where the final set of columns depends on user's input.

This is a requirement in one of our reports called "waterfall". For example, if a user selects the date range of June to Oct, then the columns are: `person`, `june_amount`, `july_amount`, `aug_amount`, `sep_amount`, and `oct_amount`. There's an example of the dynamic column SQL statement in the [README.md](https://github.com/tanin47/lilit-sql).

Since lilit-sql provides these higher-abstraction-esque capabilities (e.g. reading which column exists and defining columns dynamically), we can re-use business logic and make our engineering team more productive and efficient in maintaining a lot of SQL statements.

lilit-sql is still in its nascent. If the pain points described here resonate with you, I'd love to work with you to iterate the library to be something that fits your needs. Please don't hesitate to reach out by creating an issue on [the repository](https://github.com/tanin47/lilit-sql)!
