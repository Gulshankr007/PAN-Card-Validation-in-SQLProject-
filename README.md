# PAN-Card-Validation-in-SQLProject-

![image alt]((https://github.com/Gulshankr007/PAN-Card-Validation-in-SQLProject-/blob/da4c186e0f597462b7def70c0acb16afd48cbb77/pan%20project%20.png)

# PAN Number Validation

**Repository:** PAN Number Validation Project

---

## 📌 Project Overview

This repository contains SQL-based data cleaning and validation logic for Indian Permanent Account Numbers (PAN). The goal is to:

* Clean and standardize raw PAN inputs.
* Validate each PAN against the official format `AAAAA1234A`.
* Flag PANs as **Valid** or **Invalid** using business rules (adjacent repetition, sequential characters, etc.).
* Produce a short summary reporting totals for processed, valid, invalid and missing/incomplete PANs.

---

## 🔧 Table: `stg_pan_no_dataset`

Create the staging table to hold raw PAN values:

```sql
create table stg_pan_no_dataset
(
    pan_number text
);

select * from stg_pan_no_dataset;
```

---

## 🧹 Data cleaning & exploratory checks

Use these queries to inspect and clean the raw data.

* **Identify missing values**

```sql
-- Identify and handle missing data:
select * from stg_pan_no_dataset where pan_number is null;
```

* **Check for duplicates**

```sql
-- Check for duplicates
select pan_number, count(1)
from stg_pan_no_dataset
group by pan_number
having count(1) > 1;
```

* **Leading / trailing spaces**

```sql
-- Handle leading/trailing spaces
select * from stg_pan_no_dataset where pan_number <> trim(pan_number);
```

* **Letter case correction**

```sql
-- Correct letter case:
select * from stg_pan_no_dataset where pan_number <> upper(pan_number);
```

* **Produce cleaned pan list (distinct & standardized)**

```sql
-- Cleaned Pan Numbers:
select distinct upper(trim(pan_number)) as pan_number
from stg_pan_no_dataset
where pan_number is not null
and trim(pan_number) <> '';
```

---

## ⚙️ Validation helper functions (PL/pgSQL)

These helper functions implement the custom rules required by the problem statement.

* `fn_check_adjacent_repetition(p_str text)` — returns `true` if any adjacent characters are identical (invalid rule).

```sql
create or replace function fn_check_adjacent_repetition(p_str text)
returns boolean
language plpgsql
as $$
begin
    for i in 1 .. (length(p_str) - 1)
    loop
        if substring(p_str, i, 1) = substring(p_str, i+1, 1)
        then
            return true;
        end if;
    end loop;
    return false;
end;
$$;
```

* `fn_check_sequence(p_str text)` — returns `true` if characters form a strict ascending sequence (e.g., `ABCDE` or `1234`).

```sql
CREATE or replace function fn_check_sequence(p_str text)
returns boolean
language plpgsql
as $$
begin
    for i in 1 .. (length(p_str) - 1)
    loop
        if ascii(substring(p_str, i+1, 1)) - ascii(substring(p_str, i, 1)) <> 1
        then
            return false;
        end if;
    end loop;
    return true;
end;
$$;
```

---

## ✅ Valid / ❌ Invalid PAN categorization (view)

Create a view that classifies each cleaned PAN as `Valid PAN` or `Invalid PAN` using the helper functions and regular expression matching.

```sql
create or replace view vw_valid_invalid_pans
as
with cte_cleaned_pan as
    (select distinct upper(trim(pan_number)) as pan_number
     from stg_pan_no_dataset
     where pan_number is not null
       and TRIM(pan_number) <> ''),
cte_valid_pan as
    (select *
     from cte_cleaned_pan
     where fn_check_adjacent_repetition(pan_number) = 'false'
       and fn_check_sequence(substring(pan_number,1,5)) = 'false'
       and fn_check_sequence(substring(pan_number,6,4)) = 'false'
       and pan_number ~ '^[A-Z]{5}[0-9]{4}[A-Z]$')
select cln.pan_number
, case when vld.pan_number is null
            then 'Invalid PAN'
       else 'Valid PAN'
  end as status
from cte_cleaned_pan cln
left join cte_valid_pan vld on vld.pan_number = cln.pan_number;
```

---

## 📊 Summary report

Run this query to get totals for processed, valid, invalid and missing/incomplete PANs.

```sql
with cte as
    (SELECT
        (SELECT COUNT(*) FROM stg_pan_no_dataset) AS total_processed_records,
        COUNT(*) FILTER (WHERE vw.status = 'Valid PAN') AS total_valid_pans,
        COUNT(*) FILTER (WHERE vw.status = 'Invalid PAN') AS total_invalid_pans
    from  vw_valid_invalid_pans vw)
select total_processed_records, total_valid_pans, total_invalid_pans
, total_processed_records - (total_valid_pans+total_invalid_pans) as missing_incomplete_PANS
from cte;
```

---

## 📝 Notes & assumptions

* PAN format enforced: 10 characters, pattern `^[A-Z]{5}[0-9]{4}[A-Z]$`.
* Leading/trailing whitespace and case are normalized with `trim()` and `upper()`.
* `fn_check_adjacent_repetition` and `fn_check_sequence` detect invalid patterns as described in the problem statement.
* Rows with `NULL` or empty PANs are counted as missing/incomplete.

---

## ✨ Beautiful bullet-point checklist (with icons)

* ✅ **Table created:** `stg_pan_no_dataset` for raw data ingestion.
* 🧼 **Cleaned:** Trimmed whitespace & upper-cased PANs.
* 🔎 **Checked:** Missing values and duplicates.
* 🧩 **Validated:** Format, adjacency and sequence rules applied.
* 📑 **Categorised:** `Valid PAN` / `Invalid PAN` via view `vw_valid_invalid_pans`.
* 📊 **Reported:** Summary of processed / valid / invalid / missing counts.

---

## ▶️ How to run

1. Load raw PANs into `stg_pan_no_dataset` (CSV import, ETL, or manual insert).
2. Create the helper functions and the view in PostgreSQL (run the SQL blocks above).
3. Run the summary query to produce the final counts.

---

## 🎥 Reference

* Example video / walkthrough: [https://www.youtube.com/watch?v=J1vlhH5LFY8](https://www.youtube.com/watch?v=J1vlhH5LFY8) (techTFQ)

---

## 🧩 Contribution

Feel free to open an issue or PR if you want additional features (Python implementation, unit tests, sample dataset, or integration with GitHub Actions).

---

*Generated by: PAN Number Validation — SQL-first implementation.*
