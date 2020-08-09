SQL Code Examples
================

The following query identifies a constituent's membership in different constituent groups, pulling only constituents groups that match a specific naming pattern. It outputs a table with constituent group names as columns and each constituent as rows with a 1 or 0 to identify their membership in each constituent group.

``` r
CREATE TEMPORARY TABLE {{tmp("results")}} (
  cons_id INT
) as select cons_id from cons_table;


{% set loe_actions = eval_sql("SELECT cons_group_table from cons_groups where lower(name) LIKE '%loe action - %'") %}

ALTER TABLE {{tmp("results")}} 
{% for l in loe_actions%} 
    ADD `{{get_group_name(l.cons_group_table) | slice(0,63) | trim}}` char(1){% if not loop.last %},{%endif %}
{%endfor%};    
UPDATE {{tmp("results")}} 
SET {%for l in loe_actions%} `{{get_group_name(l.cons_group_table) | slice(0,63) | trim}}` = 0{% if not loop.last %},{% endif %} 
{%endfor%};

{%for l in loe_actions%} 
    UPDATE {{tmp("results")}}
    SET {{tmp("results")}}.`{{get_group_name(l.cons_group_table) | slice(0,63) | trim}}` = 1 
    WHERE {{tmp("results")}}.cons_id in (select cons_id from `{{l.cons_group_table}}`);
{%endfor%};


SELECT * FROM {{tmp("results")}}
```

This query pulls form signups through email for signups from 2018 through 2020. The output is a sum of total and unique signups (by email address) for each mailing, signup form, and source and subsource code.

``` r
select 
  m.mailing_id, 
  m.mailing_name, 
  sf.signup_form_id, 
  sf.signup_form_name,
  cs.source, 
  cs.subsource,
  count(*) as total_signups, 
  count(DISTINCT ce.email) as unique_signups
from 
  cons_signup cs 
  join signup_form sf using (signup_form_id) 
  join mailing_link_table ml on ml.value = cs.source 
  join mailing m using (mailing_id) 
  join cons_action ca on ca.cons_signup_id = cs.cons_signup_id
  join cons_email ce on ce.cons_id = ca.cons_id 
  where ce.is_primary = 1 
  and cs.create_dt >= '2018-01-01 00:00:00' 
  AND cs.create_dt <= '2020-12-31 23:59:59' 
group by 
  m.mailing_id, 
  m.mailing_name, 
  sf.signup_form_id, 
  sf.signup_form_name,
  cas.source, 
  cas.subsource
order by
unique_signups desc;
```
