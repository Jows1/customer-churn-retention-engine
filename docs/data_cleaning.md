# Data cleaning

How the raw Online Retail II file became the 805,549-row **Clean Data** table, why each rule exists, the known issues left in the data, and how to rebuild the whole pipeline in Power Query.

[← Back to README](../README.md)

## Raw data profile

The source file has two sheets, *Year 2009-2010* and *Year 2010-2011*, with the same eight columns.

| Column | Type | Notes |
| --- | --- | --- |
| Invoice | Text | 6-digit number; a leading "C" marks a cancellation, a leading "A" a bad-debt adjustment |
| StockCode | Text | Product code; a trailing letter marks a variant (e.g. `85099B`); some codes are not products (see below) |
| Description | Text | Product name |
| Quantity | Whole number | Negative for returns and cancellations |
| InvoiceDate | Date and time | 1 Dec 2009 07:45 to 9 Dec 2011 12:50 |
| Price | Decimal | Unit price in pounds sterling |
| Customer ID | Whole number | 5-digit ID; blank for guest checkouts |
| Country | Text | 43 countries |

| Issue found | Raw rows affected |
| --- | --- |
| Blank Customer ID | 243,007 |
| Quantity zero or negative | 22,950 |
| Price zero or negative | 6,207 |
| Invoice starts with "C" | 19,494 |
| Invoice starts with "A" | 6 |
| Exact duplicate rows | 34,335 |

## Cleaning rules

| Step | Rule | Why | Rows remaining |
| --- | --- | --- | --- |
| 1 | Append both yearly sheets | One continuous two-year history | 1,067,371 |
| 2 | Keep rows with a Customer ID | Cohorts and RFM need to know who bought | 824,364 |
| 3 | Keep Quantity > 0 | Returns would be counted as purchases and distort spend | 805,620 |
| 4 | Keep Price > 0 | Free items and adjustments aren't real revenue | 805,549 |
| 5 | Drop invoices starting with "C" | Cancelled orders aren't purchases | 805,549 |

**Why step 5 removes nothing.** Every cancelled invoice is recorded with a negative quantity, so step 3 already removed them all. Step 5 stays as a safeguard in case a future data load records a cancellation differently. The 6 bad-debt ("A") invoices have no Customer ID, so step 2 removes them.

**Order matters.** Filtering happens before any calculation. If a return or cancellation slipped through, it could become a customer's "first purchase" or "last purchase" and shift their cohort or recency.

## Calculated columns

| Column | Formula | Example |
| --- | --- | --- |
| Revenue | Quantity × Price | 12 × £6.95 = £83.40 |
| InvoiceMonth | First day of the purchase month | 2009-12-01 07:45 → 1 Dec 2009 |
| FirstPurchaseMonth | Earliest InvoiceMonth for that customer | Customer 13085 → Dec 2009 |
| CohortIndex | Months from FirstPurchaseMonth to InvoiceMonth, plus 1 | Jan 2010 purchase by a Dec 2009 customer → 2 |

FirstPurchaseMonth is calculated once per customer on the **Customer Cohort Lookup** sheet (5,878 rows) and looked up by each transaction. Calculating it directly on each of the 805,549 rows would scan the whole table once per row, which is too slow for Excel.

## Result

| Measure | Value |
| --- | --- |
| Transaction lines | 805,549 |
| Customers | 5,878 |
| Invoices | 36,969 |
| Revenue | £17,743,429.18 |
| Date range | 1 Dec 2009 to 9 Dec 2011 |

## Known issues kept in the data

**Duplicate rows.** 26,124 cleaned rows (3.2%) are exact copies of another row: same invoice, product, quantity, time and price. Together they account for £368,625, or 2.1% of revenue. Some may be genuine repeat scans of the same item, but most are likely recording duplicates.

They were kept in this version. Their effect is limited:

- **No effect** on cohorts, retention or Frequency, which count distinct customers and distinct invoices.
- **Small upward effect** on Revenue and Monetary values, about 2% overall.

Removing them is the most likely improvement for a next version. It reduces the table to 779,425 rows and revenue to £17,374,804.

**Non-product stock codes.** A few codes are charges or adjustments rather than products. They were kept because each is a real amount the customer paid, except the test rows.

| Code | Meaning | Rows |
| --- | --- | --- |
| POST | Postage | 1,838 |
| M | Manual entry | 709 |
| C2 | Carriage | 253 |
| BANK CHARGES | Bank fee | 32 |
| ADJUST, ADJUST2 | Manual adjustment | 35 |
| PADS | Padding item | 17 |
| DOT | Dotcom postage | 16 |
| D | Discount line | 5 |
| TEST001, TEST002 | Test transactions | 10 |

These codes don't affect customer counts. A product-level analysis, such as best-selling items, should exclude them.

---

## Rebuilding the pipeline in Power Query

The shipped workbook was produced with Python (pandas) using exactly the rules above. This section rebuilds the same result natively in Excel with Power Query, ending with the **Clean Data** and **Customer Cohort Lookup** sheets.

Build it in a **new, empty workbook**, not the portfolio workbook. The existing pivots and formulas depend on the current tables, and replacing them would break those links.

The pipeline uses three queries:

| Query | Loads to | Role |
| --- | --- | --- |
| `Transactions` | Connection only | Imports, combines, filters and adds Revenue and InvoiceMonth |
| `CustomerCohort` | Sheet: Customer Cohort Lookup | One row per customer with FirstPurchaseMonth |
| `CleanData` | Sheet: Clean Data | Transactions plus FirstPurchaseMonth and CohortIndex |

### Part 1: Import and combine

1. **Connect to the file.** Data → Get Data → From File → From Excel Workbook. Pick `online_retail_II.xlsx`.
2. **Select both sheets.** In the Navigator, tick *Select multiple items*, tick *Year 2009-2010* and *Year 2010-2011*, then click **Transform Data**. The Power Query Editor opens with two queries.
3. **Fix the column types in both queries.** This is the step people most often get wrong. Power Query guesses types from the first 200 rows, which are all plain numbers, so it may type Invoice and StockCode as whole numbers. Every "C" invoice and every letter-suffixed stock code would then become an **Error**. In each query, click the type icon on these column headers and set:

    | Column | Type |
    | --- | --- |
    | Invoice | Text |
    | StockCode | Text |
    | Quantity | Whole Number |
    | InvoiceDate | Date/Time |
    | Price | Decimal Number |
    | Customer ID | Whole Number |

    If a *Change Column Type* prompt appears, choose **Replace current**.

4. **Append the years.** Select the 2009-2010 query, then Home → Append Queries → **Append Queries as New**. Choose *Two tables*, with the 2010-2011 query as the second table. Rename the new query `Transactions` in the Query Settings pane.

### Part 2: Filter

Apply each filter to the `Transactions` query using the column header dropdowns. Each one adds a step under Applied Steps.

5. **Customer ID:** dropdown → untick **(null)**.
6. **Quantity:** dropdown → Number Filters → **Greater Than** → `0`.
7. **Price:** dropdown → Number Filters → **Greater Than** → `0`.
8. **Invoice:** dropdown → Text Filters → **Does Not Begin With** → `C`. Keep it upper-case; Power Query text filters are case-sensitive.

### Part 3: Add Revenue and InvoiceMonth

9. **Revenue.** Add Column → Custom Column. Name it `Revenue`, formula `[Quantity] * [Price]`. Then set its type to Decimal Number.
10. **InvoiceMonth.** Add Column → Custom Column. Name it `InvoiceMonth`, formula:

    ```
    Date.StartOfMonth(DateTime.Date([InvoiceDate]))
    ```

    Set its type to Date.

### Part 4: Build the customer cohort table

11. **Reference Transactions.** In the Queries pane, right-click `Transactions` → **Reference**. Rename the new query `CustomerCohort`.
12. **Group by customer.** Home → Group By. Choose *Basic*, group by **Customer ID**. New column name `FirstPurchaseMonth`, operation **Min**, column **InvoiceMonth**.
13. **Sort.** Customer ID dropdown → Sort Ascending. You should now have one row per customer.

### Part 5: Build the clean data table

14. **Reference Transactions again.** Right-click `Transactions` → Reference. Rename this query `CleanData`.
15. **Merge in the first purchase month.** Home → Merge Queries. Top table `CleanData`, bottom table `CustomerCohort`. Click **Customer ID** in both, join kind *Left Outer*, OK.
16. **Expand the result.** Click the expand icon on the new `CustomerCohort` column, untick everything except **FirstPurchaseMonth**, and untick *Use original column name as prefix*.
17. **CohortIndex.** Add Column → Custom Column. Name it `CohortIndex`, formula:

    ```
    (Date.Year([InvoiceMonth]) - Date.Year([FirstPurchaseMonth])) * 12
      + Date.Month([InvoiceMonth]) - Date.Month([FirstPurchaseMonth]) + 1
    ```

    Set its type to Whole Number.
18. **Restore the order.** Merging can shuffle rows. Sort by **InvoiceDate**, ascending.

### Part 6: Load

19. Home → Close & Load **To…** and set each query:

    | Query | Load as |
    | --- | --- |
    | Transactions | Only Create Connection |
    | Year 2009-2010, Year 2010-2011 | Only Create Connection |
    | CustomerCohort | Table, New worksheet |
    | CleanData | Table, New worksheet |

    To change one query's load setting later, right-click it in the Queries & Connections pane → **Load To…**.
20. **Rename the sheets** to `Customer Cohort Lookup` and `Clean Data`.

### Optional: remove duplicates

To remove the duplicate rows described under Known issues, open `Transactions` and select every column (click the first header, Shift+click the last). Then Home → Remove Rows → **Remove Duplicates**. Place this step after the Price filter and before the custom columns. The row count drops to 779,425, so update any figures you quote.

### Full query code

To check your work, or skip the clicking, open each query and use Home → **Advanced Editor**, then compare with the code below. Change the file path to where your file is saved.

**Transactions**

```
let
    Source = Excel.Workbook(File.Contents("C:\Data\online_retail_II.xlsx"), null, true),
    Year1 = Table.PromoteHeaders(Source{[Item="Year 2009-2010", Kind="Sheet"]}[Data], [PromoteAllScalars=true]),
    Year2 = Table.PromoteHeaders(Source{[Item="Year 2010-2011", Kind="Sheet"]}[Data], [PromoteAllScalars=true]),
    Combined = Table.Combine({Year1, Year2}),
    Typed = Table.TransformColumnTypes(Combined, {
        {"Invoice", type text}, {"StockCode", type text}, {"Description", type text},
        {"Quantity", Int64.Type}, {"InvoiceDate", type datetime}, {"Price", type number},
        {"Customer ID", Int64.Type}, {"Country", type text}}),
    HasCustomer = Table.SelectRows(Typed, each [Customer ID] <> null),
    NoReturns = Table.SelectRows(HasCustomer, each [Quantity] > 0),
    NoFreeItems = Table.SelectRows(NoReturns, each [Price] > 0),
    NoCancelled = Table.SelectRows(NoFreeItems, each not Text.StartsWith([Invoice], "C")),
    AddRevenue = Table.AddColumn(NoCancelled, "Revenue", each [Quantity] * [Price], type number),
    AddInvoiceMonth = Table.AddColumn(AddRevenue, "InvoiceMonth",
        each Date.StartOfMonth(DateTime.Date([InvoiceDate])), type date)
in
    AddInvoiceMonth
```

The clicked version spreads the import across three queries, while this one does it in a single query. The result is identical.

**CustomerCohort**

```
let
    Source = Transactions,
    Grouped = Table.Group(Source, {"Customer ID"},
        {{"FirstPurchaseMonth", each List.Min([InvoiceMonth]), type date}}),
    Sorted = Table.Sort(Grouped, {{"Customer ID", Order.Ascending}})
in
    Sorted
```

**CleanData**

```
let
    Source = Transactions,
    Merged = Table.NestedJoin(Source, {"Customer ID"}, CustomerCohort, {"Customer ID"},
        "Cohort", JoinKind.LeftOuter),
    Expanded = Table.ExpandTableColumn(Merged, "Cohort", {"FirstPurchaseMonth"}),
    AddCohortIndex = Table.AddColumn(Expanded, "CohortIndex",
        each (Date.Year([InvoiceMonth]) - Date.Year([FirstPurchaseMonth])) * 12
           + Date.Month([InvoiceMonth]) - Date.Month([FirstPurchaseMonth]) + 1,
        Int64.Type),
    Sorted = Table.Sort(AddCohortIndex, {{"InvoiceDate", Order.Ascending}})
in
    Sorted
```

### Check your result

The editor previews only the first 1,000 rows, so check the loaded tables, not the preview. The Queries & Connections pane shows each table's row count after loading.

| Check | Expected |
| --- | --- |
| Clean Data rows | 805,549 |
| Customer Cohort Lookup rows | 5,878 |
| Sum of Revenue | £17,743,429.18 |
| Customers with FirstPurchaseMonth = Dec 2009 | 955 |
| Customer 13085: FirstPurchaseMonth | Dec 2009 |
| Customer 13085: distinct CohortIndex values | 1, 2, 15, 20 |
| Customer 12346: distinct CohortIndex values | 1, 2, 4, 7, 14 |

If Clean Data shows fewer rows than expected, the most likely cause is step 3. Invoice or StockCode typed as a number turns those rows into errors, and the filters then drop them.

### Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| Error values in Invoice or StockCode | Column typed as a number | Change the type to Text, choose *Replace current* |
| "C" invoices still present | Filter typed as lower-case `c` | Re-enter the filter as upper-case `C` |
| CohortIndex is blank | Merge matched on the wrong column | Edit the Merge step and select Customer ID in both tables |
| Refresh is slow | 1M+ source rows | Expected; a full refresh takes a minute or two |
| File path error after moving the file | Source points to the old location | Edit the Source step, or Data → Data Source Settings → Change Source |
