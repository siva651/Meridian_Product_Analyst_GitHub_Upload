# Meridian Orders API — Product Analyst Intern Take-Home

**Candidate: Chinta Durga Siva Manikanta Reddy**

## Task 1 — Documentation vs. Data

| # | Docs say | Data actually does | Impact |
|---|---|---|---|
| 1 | `has_more` tells the client whether another page exists. If it is `true`, use `next_cursor`. | `orders_page1.json` reports `has_more: false`, but `next_cursor: cur_8f2a19bd` retrieves `orders_page2.json`, containing two more orders. | **Severe:** clients following the documented flow stop early and silently omit orders. |
| 2 | `total` always equals `subtotal + tax + shipping`. | `ord_1004`: `6200 + 511 + 599 = 7310`, but `total = 6810`. | **High:** stated arithmetic invariant is broken, causing reconciliation differences. |
| 3 | All monetary amounts are integers in the currency's smallest unit. | `ord_1006` contains `44.0`, `3.63`, `5.99`, `53.62`. | **High:** a cents-based client can interpret `$53.62` as about 54 cents. |
| 4 | `status` is one of `pending`, `shipped`, `delivered`, `cancelled`. | `ord_1003.status = refunded`. | **Moderate:** clients validating the documented enum may reject or mishandle the order. |
| 5 | `customer.email` is always present. | `ord_1005.customer.email = null`. | **Moderate:** integrations that require an email can fail or create incomplete records. |
| 6 | A missing `GET /v1/orders/{id}` returns `404`. | `GET /v1/orders/ord_9999` returned `200` with `{"order": null}`. | **Moderate:** clients may treat a missing order as a successful request. |

### Most serious issue

The pagination inconsistency is the most serious because it can silently drop an entire page of valid records while the API still returns a successful response. That is particularly risky for reporting: the resulting number can look valid even though data is missing.

## Task 2 — Total Revenue

**Selected answer: $225.70.**

Calculation choices:

- I excluded `ord_1003` because it is marked `refunded`. The supplied data does not explicitly define whether the dashboard's revenue metric excludes refunded orders, so this is a stated assumption rather than a fact.
- I interpreted `ord_1006`'s decimal values as dollars and normalized `53.62` to 5,362 cents, because the payload violates the documented integer-smallest-unit format while its line item is clearly dollar-denominated.
- For `ord_1004`, I used the reported `total` of 6,810 rather than the component sum of 7,310 because the documentation defines `total` as the amount charged. This should be confirmed with engineering.

Under the supplied data, including the refunded order would produce **$328.03**; using the computed component total for `ord_1004` instead would produce **$230.70**. The exact dashboard revenue definition cannot be determined from the supplied files alone.

## Task 3A — Reply to Priya

**Subject: Re: Revenue reconciliation (TICKET-4502)**

Hi Priya,

Thanks for flagging the discrepancy. I found several API data issues that can affect the reconciliation. Most importantly, the orders endpoint reports that there are no more records one page early, even though another page is available. That means an integration following the documented pagination rule can miss orders. One order also uses dollars-and-cents formatting instead of the documented smallest-unit format, which can cause an additional difference.

Using the supplied data and excluding the refunded order, I calculate **$225.70**, subject to the revenue-treatment assumptions documented in the analysis. The API also contains an inconsistent total for one order, so I recommend confirming the dashboard's revenue definition and refund treatment before treating this as the final accounting figure.

Regards,  
Chinta Durga Siva Manikanta Reddy

## Task 3B — Bug Report

**Title:** `GET /v1/orders` returns `has_more: false` while another page exists

**Steps to reproduce**
1. Call `GET /v1/orders`.
2. Observe `has_more: false` and `next_cursor: "cur_8f2a19bd"`.
3. Call `GET /v1/orders?starting_after=cur_8f2a19bd`.
4. The second request returns two valid orders: `ord_1005` and `ord_1006`.

**Actual**
The first response says pagination is complete even though another page is retrievable.

**Expected**
When another page exists, `has_more` should be `true` and the response should provide the cursor needed to retrieve it. The pagination metadata should be internally consistent.

**Impact**
Clients implementing the documented pagination logic stop after page 1 and silently omit orders, which can produce incomplete operational and financial reports.

**Fix**
Investigate how `has_more` and `next_cursor` are generated and ensure both are derived consistently from the same pagination state. Add a regression test that verifies a multi-page dataset cannot return `has_more: false` before the final page.
