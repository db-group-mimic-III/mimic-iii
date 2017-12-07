# Chapter 21

### Adults patients age

```SQL
select  *, case  
		when t.age_at_admission > 150 then 91.4 
		else t.age_at_admission 
		END as corrected_age
from
(select j.subject_id, timestampdiff(YEAR, p.dob, j.intime) as age_at_admission
  from
  (select subject_id, min(intime) as intime
  from  icustays
  group by subject_id) j
  left join patients p on j.subject_id = p.subject_id) t
where t.age_at_admission > 15
```
### Basic data

``` SQL

select icu.subject_id,
			icu.hadm_id,
			icu.ICUSTAY_ID,
			icu.FIRST_WARDID,
			p.gender,
			a.ADMISSION_TYPE,
			case
				when a.HOSPITAL_EXPIRE_FLAG = 1 or timestampdiff(DAY, a.deathtime, a.admittime) < 30 then 'Y'
				else 'N'
				end as thirty_day_mort,
			x.corrected_age

from ICUSTAYS icu
left join patients p  on icu.subject_id = p.subject_id
left join ADMISSIONS a on icu.HADM_ID = a.HADM_ID
# Add the age of the patients
left join (select  t.subject_id, 
		   case  
			when t.age_at_admission > 150 then 91.4 
			else t.age_at_admission 
			END as corrected_age
		from
			(select j.subject_id, timestampdiff(YEAR, p.dob, j.intime) as age_at_admission
			from
				(select subject_id, min(intime) as intime
				from  icustays
				group by subject_id) j
			left join patients p on j.subject_id = p.subject_id) t
			where t.age_at_admission > 15) x on icu.subject_id = x.subject_id

# Only patients over 15 years old
where icu.subject_id in (select  subject_id
			from
				(select j.subject_id, timestampdiff(YEAR, p.dob, j.intime) as age_at_admission
				from
					(select subject_id, min(intime) as intime
					from  icustays
					group by subject_id) j
				left join patients p on j.subject_id = p.subject_id) t
			where t.age_at_admission > 15)
;

```
