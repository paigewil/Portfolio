SQL Code Examples
================

The following query identifies a constituent's membership in different constituent groups, pulling only constituents groups that match a specific naming pattern. It outputs a table with constituent group names as columns and each constituent as rows with a 1 or 0 to identify their membership in the constituent group.

``` r
CREATE TEMPORARY TABLE {{tmp("results")}} (
  cons_id INT
) as select cons_id from cons_table;


{% set specific_cgs = eval_sql("SELECT cons_group_membership_table from cons_groups where lower(name) LIKE '%example - %'") %} # pulling tables of constituent groups of interest

ALTER TABLE {{tmp("results")}} 
{% for l in specific_cgs %} 
    ADD `{{get_group_name(l.cons_group_membership_table) | slice(0,63) | trim}}` char(1){% if not loop.last %},{% endif %}
{% endfor %}; # adding columns
UPDATE {{tmp("results")}} 
SET {% for l in specific_cgs %} `{{get_group_name(l.cons_group_membership_table) | slice(0,63) | trim}}` = 0{% if not loop.last %},{% endif %} 
{% endfor %}; # setting default value (no membership in constituent group)

{% for l in specific_cgs %} 
    UPDATE {{tmp("results")}}
    SET {{tmp("results")}}.`{{get_group_name(l.cons_group_membership_table) | slice(0,63) | trim}}` = 1 
    WHERE {{tmp("results")}}.cons_id in (select cons_id from `{{l.cons_group_membership_table}}`); 
{% endfor %}; #For constituents with membership in the constituent group, assign value of 1


SELECT * FROM {{tmp("results")}}
```

This query pulls form signups through email for signups from 2018 through 2020. The output is a sum of total and unique signups (by email address) for each mailing, signup form, and source and subsource code.

``` r
select 
  m.mailing_id, 
  m.mailing_name, 
  sf.signup_form_id, 
  sf.signup_form_name,
  s.source, 
  s.subsource,
  count(*) as total_signups, 
  count(DISTINCT ce.email) as unique_signups
from 
  signup s 
  join signup_form sf using (signup_form_id) # to pull signup form name
  join mailing_link_table ml on ml.value = s.source # to match form source to mailing
  join mailing m using (mailing_id) # to pull email name
  join cons_signup cs on cs.cons_signup_id = s.cons_signup_id #to match signup to constituent record
  join cons_email ce on ce.cons_id = cs.cons_id # to pull email address of constituent record
  where ce.is_primary = 1 
  and s.create_dt >= '2018-01-01 00:00:00' 
  AND s.create_dt <= '2020-12-31 23:59:59' 
group by 
  m.mailing_id, 
  m.mailing_name, 
  sf.signup_form_id, 
  sf.signup_form_name,
  s.source, 
  s.subsource
order by
unique_signups desc;
```

This query pulls the constituent record IDs for all constituents who have donated more than once in the last 18 months.

``` r
select (DISTINCT cons_id)

from (
  select 
  cons_id,
    count(DISTINCT contribution_id) records
  from cons_contribution
  where transaction_dt >= date_sub(now(), interval 18 month)
  group by 1
  having records > 1
) i
```
