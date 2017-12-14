Supplemental Queries:
Listed below are queries that we created. They are either precursor queries that were required to formulate our more advanced queries, or advanced queries that ended up being inconclusinve or did not do what we expected them to. These are important to our learning process, but we kept them separate to for code clairity. 

### Determination of biological variables
These queries were used to find the proper item id and lab id values for our SIRS query. These relate to part 2 of our project and taught us a lot about "write in" systems and the associated pros and cons. Ultimately, Mario decided which values to use, this was based off of his clinical expertise.
##### Temp measures
``` SQL
 Select *
 from D_items
 where label like "temp%"
 ;
 ```
 42 rows start with "temp" 2 basic groups of temperature and temporary. Some temperature ones are appreviated temp though.
   there is rectal, oral, skin temp and a few others. Mario, which ones are most important?
 
 ##### Heart Rate measures
 ```SQL
 Select *
 from D_ITEMS 
 where label = 'Heart Rate'
 ;
 ```
 2 rows matter, the difference is carevue vs metavision
 ItemID labels are "211" and "220045"
 
 
 ##### W_blood cell count 
 ``` SQL
 select *
 from d_items
 where label like 'wb%' or label like 'white%'
 ;
 ```
 Queries 10 values, 6 seem to be potential options, the others I don't understand.
 
 
 ##### breath related measurements
 ``` SQL
 Select *
 from d_items
 where label like 'bre%' or label like '%CO2' or label like '%vent%'
 order by label
 ;
 ```
 Breath Rate is an easy one
 Not sure how to handleor look at the ventilator related values. I would think patients on a
 ventilator would be at an increased risk for sepsis but is BPM measured in them? 
 
 Related to CO2 measurements, what is the difference between PaCO2, pCO2, Total CO2, VCO2, Venouse CO2?

### Analysis of the biological variables
##### WBC
```SQL
select max(valuenum), min(valuenum), avg(VALUENUM), count(valuenum), itemid
from chartevents
where itemid in( 861, 1542, 220546,1127, 4200, 3834)
group by ITEMID;
```

##### Arterial PCO2 gases
```SQL
select max(valuenum), min(valuenum), avg(VALUENUM), count(valuenum), itemid
from chartevents
where itemid in( 778,  3784, 3835)
group by ITEMID;


```
#### Create intervention intervals
##### Microbiology
The second half of Sepsis is having an infection. Physicians generally administer an antiobiotic proactively while a microbiolgy lab is being performed to confirm a bacterial presence. While a positive microbiology result would be a true positive, the extended length of time required to perform the test makes it difficult to practically create a sensitve algorithm. Additionally, we cannot be sure whether the timestamp was for when the test was performed, or when the result came back. Since a microbiology report takes over 24 hrs to process, this difference was important. Additionally, Charttime was not consistently recorded in the microbiology table. This also prevented us from depending on these results for our algorithm. 
``` SQL
# Microbiology time is difficult to interpret 
select hadm_id, chartdate, charttime, SPEC_itemid, SPEC_TYPE_DESC, org_itemid
from microbiologyevents
# Retrieve only patients with sepsis
where hadm_id in (
	select hadm_id
	from sepsis_patients where subject_id = 10188
	)
# Exclude MRS screenng and other screening, they are routine in hospitals
and  SPEC_TYPE_DESC not in (
	select distinct SPEC_TYPE_DESC
	from microbiologyevents
	where SPEC_TYPE_DESC like '%screen%' 
	and SPEC_TYPE_DESC != 'Rapid Respiratory Viral Screen & Culture'
````

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
