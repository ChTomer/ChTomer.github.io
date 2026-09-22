---
title: "Building a Data Warehouse That Can't Lie to You"
date: 2026-09-09
layout: single
categories: [projects]
tags: [sql, data-engineering, data-warehouse, power-bi, dax, etl]
author_profile: true
read_time: true

excerpt: "How a single wrong JOIN taught me more about data modeling than any tutorial - and why I rebuilt an entire warehouse around three fact tables instead of one."

header:
  teaser: /assets/images/5_Delivery_Promise___Satisfactions.png
---

# Building a Data Warehouse That Can't Lie to You

<img src="/assets/images/5_Delivery_Promise___Satisfactions.png" style="width:100%;">
<p style="font-size: 0.85em; text-align: left;">
From raw CSVs to a five-page Power BI dashboard - the full journey of one e-commerce dataset.</p>

**Posted by Tomer Choresh** <br>
*Published on: September 9, 2026*

---

## Why I Built This

Most of my previous projects started with a CSV file and ended with a chart. This one starts with a CSV file too - but everything in between is different. Instead of jumping straight to `pandas.read_csv()` and modeling, I wanted to build the layer that usually sits *before* the analysis: the pipeline that takes messy operational data and turns it into something a dashboard can trust.

So I picked the [Olist Brazilian E-Commerce dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) - about 100,000 real orders from a Brazilian marketplace, spanning customers, sellers, products, payments, and reviews - and set out to build a full Data Warehouse from scratch: SQL Server on the backend, Power BI on the front, and nothing skipped in between.

I did not expect the most interesting part of the project to be a bug I found in my *own* design, three days in.

---

## Data and Tools

| Layer | Purpose |
|---|---|
| `olist_db` | Raw 1:1 mirror of the Kaggle CSVs - no cleaning, no logic |
| `olist_STG` | Cleaned copy - trimmed strings, translated categories, no grain changes |
| `olist_DWH` | The dimensional model that actually powers the dashboard |

Stack: **SQL Server** (running in a Windows VM on my Mac - more on that later), **Power BI Desktop** in Import mode, and **DAX** for everything from simple sums to unbiased averages.

Everything is scripted - all 15 pipeline files, numbered in run order, live in the [GitHub repo](https://github.com/ChTomer/olist-capstone-dwh). No manual clicking, no "trust me, I ran it once."

---

## The Bug I Almost Shipped

My first version of the warehouse looked like every Star Schema tutorial: one `Fact_Sales` table at order-item grain, with customer, product, and seller dimensions hanging off it. Clean, textbook, done.

Then I needed to add payment and review data, and I did what felt natural: `LEFT JOIN` the order-level payment table onto my item-level fact table.

Here's the problem, made small enough to see at a glance. Say one order has 3 items and 1 payment:

```
order_id | order_item_id | payment_value
A100     | 1              | 90
A100     | 2              | 90
A100     | 3              | 90
```

`SUM(payment_value)` now returns **270** for a payment that was actually **90**. The payment row got duplicated across every item row it touched - a classic **fan-out**. The same thing happened with reviews: an order with 3 items and 1 five-star review would count that single review three times, quietly dragging the average review score toward whichever orders happened to have the most items.

I only caught it because a total looked *slightly* too high. If it had looked plausible, it would have shipped.

> *Takeaway: fan-out doesn't announce itself. It just makes your numbers wrong by an amount small enough to not look wrong.*

---

## The Fix: Fact Constellation

The tempting fix was to patch the measures - use `DISTINCTCOUNT` here, add a filter there. It would have worked. It also would have meant that every future person (including future me) writing a new DAX measure against this model would need to *remember* those unwritten rules, forever, with no warning when they forgot.

So instead I rebuilt the model as a **Fact Constellation** (a.k.a. Galaxy Schema): three fact tables, each pinned to its own natural grain, sharing dimensions instead of sharing rows.

| Fact | Grain |
|---|---|
| `Fact_Order_Items` | order_id + order_item_id |
| `Fact_Payments` | order_id + payment_sequential |
| `Fact_Reviews` | review_id + order_id |

Now a wrong join doesn't silently inflate a number - it just doesn't work, because the grains don't line up. The mistake becomes structural instead of a matter of institutional memory. That trade-off (more tables, more upfront design, in exchange for a model that fails loudly instead of quietly) is the single decision I'd point to as the core of this project.

---

## More Grain Surprises Along the Way

Once I started treating "grain" as the first question for every table, I kept finding smaller versions of the same problem:

- **`customer_id` vs. `customer_unique_id`** - 99,441 vs. 96,096. The first is order-scoped (a new ID every time someone orders), the second is the actual person. Building `Dim_Customer` on the wrong one would have quietly inflated the customer count by 3,345 phantom people.
- **A circular relationship in Power BI** - `Dim_Order` originally held a date key that created a closed loop between two dimension tables through the fact table. Power BI doesn't error on this; it just silently disables one of the two paths. I only found it by testing a measure that should have responded to a filter and didn't.
- **The same column, stored twice** - delivery-delay metrics briefly existed in both `Dim_Order` and `Fact_Order_Items`, and the two disagreed on how many `NULL`s they had. One of them had to go; I kept the one at the grain the ETL pipeline actually respects.

None of these were dramatic on their own. Together, they convinced me that most "the dashboard shows a weird number" bugs are grain bugs wearing a disguise.

---

## Touring the Dashboard

The final Power BI dashboard has five pages, each answering one of the business questions I started with.

**Overview** - what's driving revenue, broken down into eight rolled-up category groups (from 74 raw ones).

<img src="/assets/images/1_Overview.png" style="width:100%;">

**Where the Time Goes** - not just *whether* an order is late, but *where* the time is lost. I split total delivery time into Handling (seller's responsibility, purchase → carrier hand-off) and Transit (carrier's responsibility, carrier → customer) - so "late" stops being one number and starts pointing at who's actually accountable.

<img src="/assets/images/2_Where_the_Time_Goes.png" style="width:100%;">

**Sellers & Geography** - a single map and chart that toggles between a customer view and a seller view via Bookmarks, so "where is the business happening" doesn't collapse two different questions into one confusing map.

<img src="/assets/images/3A_Seller___Geography.png" style="width:100%;">

**Payments & Customer Voice** - how people paid (credit card dominates, unsurprisingly) and what they said about it.

<img src="/assets/images/4_Payments___Customer_Voice.png" style="width:100%;">

**Delivery Promise & Satisfaction** - the page I'm most attached to. More on it below.

<img src="/assets/images/5_Delivery_Promise___Satisfactions.png" style="width:100%;">

---

## The Delivery Cliff

This is the chart that made the whole "grain first" philosophy feel worth it. Once the model could safely cross between the delivery-timing fact and the review fact without a fragile join, I could finally ask: *does satisfaction fall off gradually with delay, or does it fall off a cliff?*

It's a cliff. Average review score sits around 4.3 for every early or on-time order, and once an order crosses zero days late, it drops sharply and keeps falling. Across the delivered population, being late costs an average of **2.02 points** on a 5-point scale (`Points Lost to Lateness`), even though only **6.77%** of delivered orders were late at all.

Breaking that down by how late an order actually was tells the same story with more nuance: an order 5-9 days into its delivery window scores 4.37 if the promise was kept vs. 3.59 if it was broken - and that "broken promise" gap holds at a similar size (roughly 0.7-1.0 points) whether the wait was short or long. It's not the wait that hurts the score. It's the broken promise.

---

## What I Found Chasing My Own Curiosity

Past the dashboard requirements, I let myself go down a few rabbit holes:

- **3.12%** of customers ordered more than once. Repeat customers came back after a median of 28 days - but a long right tail (average 80 days) means most people who return, return fast.
- The top 20% of sellers by revenue account for **82.7%** of total revenue - almost exactly the textbook 80/20 split.
- I tested whether Brazil's "Mother's Day" shopping spike was a real, repeatable pattern the way Black Friday clearly was. 2018 showed a 57% jump in the two weeks before the holiday; 2017 showed a slight *decrease* over the same window. One year isn't a pattern, so I documented it as tested-but-not-confirmed instead of writing it up as a finding. Being wrong quietly is worse than being uncertain out loud.

---

## Key Takeaways

- **Grain first, always.** Every real bug in this project - fan-out, inflated customer counts, the circular relationship - traced back to getting the grain wrong somewhere.
- **Structural safety beats convention.** A model that can't be misused is worth more than a model with a well-documented list of ways not to misuse it.
- **A number that looks slightly too high is worth ten minutes of suspicion.** The fan-out bug never looked absurd. It just looked a little off.
- **Running SQL Server inside a Windows VM on a Mac** turned out to be its own small adventure - OneDrive sync paths, BULK INSERT permissions, and a healthy respect for keeping every step scripted so the whole thing could be rebuilt from zero without me remembering what I clicked.

---

## What's Next

The project is "done" in the sense that it's presented and published - but I already have a running list of things I'd like to revisit: extending the Bookmark-based toggle pattern to more pages, and digging further into the seller-concentration finding to see if it holds up by category, not just overall.

---

Thanks for reading. The full pipeline, DAX measures, and dashboard screenshots are on my [GitHub](https://github.com/ChTomer/olist-capstone-dwh), and I'm always happy to connect on [LinkedIn](https://www.linkedin.com/in/tomer-choresh/).
