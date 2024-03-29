## SQL query to calculate the shadow_index for Rxivist data

```sql
SELECT n.article,
(n.rank_number - 1) / (n.max_collection_rank::decimal - 1) * 100 AS shadow_index
FROM
(SELECT m.*,
RANK () OVER (
  PARTITION BY collection
  ORDER BY avg_views_rate DESC
  ) AS rank_number,
COUNT (1) OVER (
    PARTITION BY collection
    ) AS max_collection_rank
FROM
(SELECT l.article, l.collection,
(SUM(l.abstract::decimal) / MAX(l.day_diff::decimal)) AS avg_views_rate
FROM
(SELECT *
FROM
(SELECT *,
j.traffic_date - j.posted AS day_diff
FROM
(SELECT t.article, a.collection,
(TO_DATE(t.year||'-'||lpad(t.month::text, 2, '0')||'-'||'01', 'YYYY-MM-DD') + ('1 month')::interval)::date AS traffic_date,
a.posted,
t.abstract
FROM prod.article_traffic t
LEFT JOIN prod.articles a
ON a.id = t.article
WHERE t.article NOT IN
  (SELECT article
  FROM prod.publication_dates d
  )
) j
WHERE j.traffic_date > j.posted) k
WHERE k.day_diff <= 100) l
GROUP by l.article, l.collection
) m
) n
ORDER BY 1;
```

## Explanation

An article visibility index ("shadow index") is needed to sort bioRxiv preprints in order to highlight underrepresented preprints. For the MVP this is based on article views ('abstract' column in the 'articles' table).

The query converts the monthly traffic counts in the 'traffic' table into a date, adding one month to give a value of the first day of the following month. Preprints that have an identified published version ('publications_date' table) are excluded, and we are only concerned with the traffic for the first 100 days of a preprint's life.

The difference between the preprint posted date, and the last traffic data date is calculated in days, and used to calculate a daily view rate.

These rates are used to calculate a ranking by article category, and normalised to 100 (being the preprint in a particular category with the lowest view rate in its first 100 days).

## Further work

* Incorporate more fields into the index
* Is 100 days appropriate?