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
# separation by age group
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

# separation by age groups and gender
SELECT
  COUNT(*),
  CASE
    WHEN age_admission_icu >=18 AND age_admission_icu <=25 and gender = "F" THEN 'F:18-25' #41
	WHEN age_admission_icu >=18 AND age_admission_icu <=25 and gender = "M" THEN 'M:18-25' #26
    WHEN age_admission_icu >=26 AND age_admission_icu <=34 and gender = "F" THEN 'F:26-34' #77
	WHEN age_admission_icu >=26 AND age_admission_icu <=34 and gender = "M" THEN 'M:26-34' #106
    WHEN age_admission_icu >=35 AND age_admission_icu <=54 and gender = "F" THEN 'F:35-54' #465
	WHEN age_admission_icu >=35 AND age_admission_icu <=54 and gender = "M" THEN 'M:35-54' #670
	WHEN age_admission_icu >=55 AND age_admission_icu <=64 and gender = "F" THEN 'F:55-64' #507
	WHEN age_admission_icu >=55 AND age_admission_icu <=64 and gender = "M" THEN 'M:55-64' #705
    WHEN age_admission_icu >=65 AND age_admission_icu <=80 and gender = "F" THEN 'F:65-80' #871
	WHEN age_admission_icu >=65 AND age_admission_icu <=80 and gender = "M" THEN 'M:65-80' #1215
    WHEN age_admission_icu >=81 and gender = "F" THEN 'F:81+' #646
	WHEN age_admission_icu >=81 and gender = "M" THEN 'M:81+' #647
  END AS age_groups
FROM sepsis_patients s
left join patients p
on s.SUBJECT_ID = p.subject_id
group by age_groups
        ;
  # Top used drugs in sepsis patients      
Select drug, count(drug)
  from sepsis_patients s
  left join PRESCRIPTIONS p
on s.subject_id=p.subject_id
group by p.drug
order by count(drug) desc
limit 10
;
