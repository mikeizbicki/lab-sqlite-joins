# Lab: SQLite, Subqueries, and JOINs

The general consensus in industry is that it's basically never wrong to store your data in SQLite.

<img src=sqlite-meme.jpg width=300px />

This lab will review:
1. strengths of SQLite (easy to distribute databases),
1. weaknesses of SQLite (weak typing, sometimes slow, missing SQL syntax),
1. navigating unfamiliar databases, and
1. LEFT JOIN and the NOT IN operations.

## Background

We've seen before that datasets are often distributed as JSON files or CSV files.
For complicated datasets, however, this can be inconvenient.
For example, some datasets might require many CSV files to store the entire dataset.
One common use of SQLite is to distribute these datasets as a single file.
Each CSV/JSON file in the original dataset would correspond to one table in the SQLite database file.

For this lab, we will use sqlite3 to explore the "San Francisco Restaurant Health Inspections Dataset".
This dataset contains records of health inspection violations between 2013-2016 in San Francisco restaurants.
The dataset was originally prepared in CSV form by the city government,
but a Stanford social scientist reformatted it into a SQLite database for easier distribution.
You can find the SQLite dataset in Stanford's "Public Affairs Data Journalism" webpage at <http://2016.padjo.org/tutorials/sqlite-data-starterpacks/#more-info-san-francisco-restaurant-health-inspections>.

Find the URL of the sqlite dataset in the webpage above (it will end in `.sqlite`). 
Then download it with the `wget` command.
```
$ wget <url>
```

Open the file with `sqlite3` and use the `.tables` dot command to check what tables exist in the dataset.
```
$ sqlite3 sfscores.sqlite
sqlite> .tables
businesses   inspections  violations
```
Each of these tables corresponds to a CSV file that was originally distributed by the city government.

A good first step to understanding these tables is to count how many rows they have.
```
sqlite> select count(*) from businesses;
7634
sqlite> select count(*) from inspections;
27343
sqlite> select count(*) from violations;
40735
```
It makes sense that we have more inspections than businesses...
but maybe it should be concerning that we have more violations than inspections.
On average, every health inspection results in 2 violations.

Let's take a look at the violations table to get a sense of what these violations look like.
First, let's convert the output format to markdown to make it more readable.
```
sqlite> .mode markdown
```
Then let's list the first 10 rows of the table.
```
sqlite> select * from violations limit 10;
| business_id |   date   | ViolationTypeID | risk_category |                    description                     |
|-------------|----------|-----------------|---------------|----------------------------------------------------|
| 10          | 20140729 | 103129          | Moderate Risk | Insufficient hot water or running water            |
| 10          | 20140729 | 103144          | Low Risk      | Unapproved or unmaintained equipment or utensils   |
| 10          | 20140114 | 103119          | Moderate Risk | Inadequate and inaccessible handwashing facilities |
| 10          | 20140114 | 103145          | Low Risk      | Improper storage of equipment utensils or linens   |
| 10          | 20140114 | 103154          | Low Risk      | Unclean or degraded floors walls or ceilings       |
| 10          | 20160503 | 103114          | High Risk     | High risk vermin infestation                       |
| 10          | 20160503 | 103103          | High Risk     | High risk food holding temperature                 |
| 10          | 20160503 | 103144          | Low Risk      | Unapproved or unmaintained equipment or utensils   |
| 10          | 20160503 | 103103          | High Risk     | High risk food holding temperature                 |
| 10          | 20160503 | 103148          | Low Risk      | No thermometers or uncalibrated thermometers       |
```
Business 10 apparently has a lot of problems.
The "high risk vermin infestation" looks particularly bad,
and I definitely wouldn't want to eat at restaurant with rats in the kitchen.
I wonder what business this is?

To find out, we can use a simple inner join.
The `business_id` column is suggestive that there is a corresponding column in the `businesses` table.
We can verify that with the `.schema` command.
```
sqlite> .schema businesses
CREATE TABLE businesses (
	business_id INTEGER NOT NULL,
	name VARCHAR(64),
	address VARCHAR(50),
	city VARCHAR(23),
	postal_code VARCHAR(9),
	latitude FLOAT,
	longitude FLOAT,
	phone_number BIGINT,
	"TaxCode" VARCHAR(4),
	business_certificate INTEGER,
	application_date DATE,
	owner_name VARCHAR(99),
	owner_address VARCHAR(74),
	owner_city VARCHAR(22),
	owner_state VARCHAR(14),
	owner_zip VARCHAR(15)
);
```
Inspecting this schema, it looks like the `name` field contains the information we want.

The following JOIN will place the name of each business next to their violation.
```
sqlite> SELECT name, violations.*
   ...> FROM businesses
   ...> JOIN violations USING (business_id)
   ...> LIMIT 10;
|       name       | business_id |   date   | ViolationTypeID | risk_category |                    description                     |
|------------------|-------------|----------|-----------------|---------------|----------------------------------------------------|
| Tiramisu Kitchen | 10          | 20140729 | 103129          | Moderate Risk | Insufficient hot water or running water            |
| Tiramisu Kitchen | 10          | 20140729 | 103144          | Low Risk      | Unapproved or unmaintained equipment or utensils   |
| Tiramisu Kitchen | 10          | 20140114 | 103119          | Moderate Risk | Inadequate and inaccessible handwashing facilities |
| Tiramisu Kitchen | 10          | 20140114 | 103145          | Low Risk      | Improper storage of equipment utensils or linens   |
| Tiramisu Kitchen | 10          | 20140114 | 103154          | Low Risk      | Unclean or degraded floors walls or ceilings       |
| Tiramisu Kitchen | 10          | 20160503 | 103114          | High Risk     | High risk vermin infestation                       |
| Tiramisu Kitchen | 10          | 20160503 | 103103          | High Risk     | High risk food holding temperature                 |
| Tiramisu Kitchen | 10          | 20160503 | 103144          | Low Risk      | Unapproved or unmaintained equipment or utensils   |
| Tiramisu Kitchen | 10          | 20160503 | 103103          | High Risk     | High risk food holding temperature                 |
| Tiramisu Kitchen | 10          | 20160503 | 103148          | Low Risk      | No thermometers or uncalibrated thermometers       |
```
And now we know not to eat at Tiramisu Kitchen.

Also notice the behavior of the qualified `violations.*` in the column list.
This is a useful trick for quickly writing a reasonable column list when joining tables.

## A First SQLite Oddity

Let's slightly generalize the above query to add the name to every violation.
To make the results readable, we'll also order by the `business_id`.
```
SELECT name, violations.*
FROM violations
JOIN businesses USING (business_id)
ORDER BY business_id;
```
Notice anything weird about the results?

They don't seem very ordered.
For example, 9948 comes before 999 in the `business_id` column.

Why?

We will soon see that the answer is due to SQLite being *weakly typed*.
([The authors of SQLite consider this terminology derogatory and prefer the more pleasant sounding *flexibly typed*](https://www.sqlite.org/quirks.html)).
Recall that begin weakly typed means that any column can contain any type,
even if the schema states that it should contain a different type.
This is bad because it can result in incorrect behavior without giving appropriate warnings/errors.

We can use the `typeof` function to compute the type of any value.
Modify the query above to use the `typeof` function to compute the type of our `business_id` columns:
```
SELECT name, typeof(business_id), violations.*
FROM violations
JOIN businesses USING (business_id)
ORDER BY business_id;
``` 
You should see that the type of most columns is `text`.
If you re-run the `.schema businesses` command, however,
you'll see that the type of these variables should be `INTEGER`.
In this case, the change of type has a relatively mild consequence:
the ORDER BY clause sorts ASCIIbetically instead of numerically like we expected.

> **Note:**
> Recall that this dataset was prepared by a Stanford professor,
> and this dataset has basic "errors" in it like confusing integers and strings.
> One of your takeaways from this lab should be that no human is perfect.
> You therefore shouldn't trust that even highly credentialed people like Stanford professors are handling their data correctly.

In order to get the expected sorting behavior,
you need to cast the `business_id` to an `INTEGER` before sorting.
The following modified query achieves this result.
```
SELECT name, violations.*
FROM violations
JOIN businesses USING (business_id)
ORDER BY CAST(business_id AS INTEGER);
```

## Subqueries => Outer Joins

Another type of query that we might want to ask is:
has every business had a health inspection?

A natural way to do this is with the `NOT IN` clause and a subquery:
```
SELECT business_id, name
FROM businesses
WHERE business_id NOT IN (
    SELECT business_id
    FROM inspections
    WHERE business_id IS NOT NULL
    );
```
That's a lot of businesses!

How many?

Let's count them by changing the column list to `count(*)`.
```
SELECT count(*)
FROM businesses
WHERE business_id NOT IN (
    SELECT business_id
    FROM inspections
    WHERE business_id IS NOT NULL
    );
```
Notice that both queries above ran very fast.
SQLite provides a built-in timer to measure how fast queries run.
Enable it by running the dot command
```
.timing on
```
Then rerun the counting query above.
The last line of the output should look something like
```
Run Time: real 0.017 user 0.014570 sys 0.002421
```
The `real` time is the number of seconds elapsed, and the number we're interested in.
This is a fast query, running in only 17 milliseconds.

We will now rewrite this query as an equivalent LEFT JOIN.
```
SELECT count(*)
FROM businesses
LEFT JOIN inspections USING (business_id)
WHERE inspections.business_id IS NULL;
```

> **NOTE:**
> If it's not clear why the LEFT JOIN above is equivalent to the NOT IN query,
> that's okay.
> I goofed up the explanation in class, and we'll cover this transformation again next week on Tuesday.

Notice when running the LEFT JOIN query you get the same results as you did when running the NOT IN query.
These queries are 100% equivalent (by definition),
and so guaranteed to always generate the same results.
Unfortunately, the latter query is much slower.
For me, it was 12 seconds---a slowdown of roughly 1 million times!

Why?

SQLite is an *embedded database*.
This means that it is intentionally designed to be much simpler than other other database engines
in order to be *embedded* in other applications.
(Recall that SQLite [is likely the world's most widely deployed software](https://www.sqlite.org/mostdeployed.html),
and one of the reasons for this is the easy with which it can be included in other programs.)

One of the ways that SQLite is intentionally simpler is that it does not include an advanced *query optimizer* that can convert SQL expressions into more efficient forms.
Other databases (like Postgres) can easily optimize both SQL queries into equally efficient forms.
We will talk about query optimization in more detail after the midterm.

<!--
<img src=sqlite-meme2.webp width=400px>
-->

### One Final SQLite Oddity

Another weakness of SQLite is that it does not have full support for set operations.
The following query, for example, would be equivalent to the NOT IN and LEFT JOIN count queries above.
```
SELECT count(*)
FROM (
    SELECT business_id FROM businesses
    EXCEPT ALL
    SELECT business_id FROM inspections
) AS t;
```
But sqlite3 does not support the `EXCEPT ALL` operator,
and so it generates an error.

<!--
> **NOTE:**
> SQLite does in fact support the EXCEPT operator.
> And the only difference between EXCEPT and EXCEPT ALL is that the former removes duplicates.
> In this case, we expect that that `businesses` table should not have any duplicates,
> and so these operators should behave the same.
> Unfortunately, if you try running the command above `EXCEPT` instead of `EXCEPT ALL`:
> ```
> SELECT count(*)
> FROM (
>     SELECT business_id FROM businesses
>     EXCEPT
>     SELECT business_id FROM inspections
> ) AS t;
> ```
> You will get very inaccurate results.
> Nothing will be removed because the type of the `business_id` in `inspections` is `text`,
> and the type in `businesses` is `integer`.
-->

## Your Final Task

Let's pretend you're a data analysis working for the city of San Francisco.
You've been tasked with finding the number of businesses who have been inspected by the health inspector and never had a violation.
As a first pass, you've written the following SQL query.
```
SELECT count(*)
FROM inspections
JOIN businesses USING (business_id)
WHERE business_id NOT IN (
    SELECT business_id
    FROM violations
    WHERE business_id IS NOT NULL
);
```
<!--
which is equivalent to
```
SELECT count(*)
FROM inspections
JOIN businesses USING (business_id)
LEFT JOIN violations USING (business_id)
WHERE violations.business_id IS NULL;
```
-->
Your idea is that:
1. the JOIN between inspections and businesses assures that you're only capturing businesses who have been inspected, and
1. the NOT IN clause ensures that the business has not had any violations.

Unfortunately, the query above is wrong.
The query above returns the number 2183,
but the correct number is 860.

Your task is to debug and fix the query above.

Copy/paste the fixed query into sakai to receive credit for the lab.

<!--
SELECT count(DISTINCT business_id)
FROM inspections
JOIN businesses USING (business_id)
WHERE business_id NOT IN (
    SELECT business_id
    FROM violations
    WHERE business_id IS NOT NULL
);
-->
