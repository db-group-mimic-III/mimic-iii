!!! I created the sepsis_patient table using Mario's code. When counting unique HADM_ID's and subject_Id's I got a count of 5976 This is more than what sepsis summary says is our patient total !!!

````Sql
select distinct count(subject_id)
from sepsis_patients # count is 5976
;
# this gener query also has more counted than we have acual Sepsis patients. 
# 2607 Females    3369 Males
Select p.gender,count(gender)
from sepsis_patients S 
left join patients p
on s.subject_id=p.subject_id
group by gender
;

# There are multiple sub-ethnicities in this query. How what type of command to I do to
get them all to be counted as the same type? 
Select a.ethnicity,count(ethnicity)
from sepsis_patients S 
left join admissions a
on s.subject_id=a.subject_id
where ethnicity like "Asian%"
group by ethnicity
order by ethnicity
;
