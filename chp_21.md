# Chapter 21

### Adults patients

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
