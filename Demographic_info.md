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
# get them all to be counted as the same type? 
Select a.ethnicity,count(ethnicity)
from sepsis_patients S 
left join admissions a
on s.subject_id=a.subject_id
where ethnicity like "Asian%"
group by ethnicity
order by ethnicity
;

SELECT
  COUNT(*),
  CASE
    WHEN age_admission_icu>=18 AND age_admission_icu <=25 THEN '18-25' # 67
    WHEN age_admission_icu >=26 AND age_admission_icu <=34 THEN '26-34' # 183
    WHEN age_admission_icu >=35 AND age_admission_icu <=54 THEN '35-54' # 1135
    WHEN age_admission_icu >=55 AND age_admission_icu <=64 THEN '55-64'# 1212
    WHEN age_admission_icu >=65 AND age_admission_icu <=80 THEN '65-80' # 2086
    WHEN age_admission_icu >=81 THEN '81+' # 1293
  END AS age_groups
FROM sepsis_patients
group by age_groups
        ;
