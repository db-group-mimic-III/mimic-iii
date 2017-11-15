# Sepsis analysis
## Definition of sepsis
Adult patient with a suspected or comprobated source of infection

#### Systemic inflammatory response syndrome (SIRS)
Clinical manifestations (two or more)
  * Fever > 100.4 F or hypothermia < 98.6 F
  * Leukocytosis > 12,000 cells/mm, Leukopenia < 4,000 cells/mm or >10% bands
  * Tachycarida > 90
  * Hyperventilation > 20 breaths per minute or PaCO2 < 32 mmHg

### Codification of sepsis
| Code        | Name of the code | Requires             |
| ------------|------------------| ---------------------|
| 995.90      | Unspecified SIRS | Underlying condition |
| 995.91      | Sepsis (SIRS due to infectious process without organ dysfunction)           |  Underlying condition, then sepsis code  |
| 995.92      | Severe sepsis (SIRS due to infectious process with organ dysfunction)    |  Underlying condition, severe sepsis code, then organ failure  |


## Exploratory analysis
###### Count patients with sepsis `1,198` or severe sepsis `3,560`
```SQL
select short_title, long_title, j.icd9_code, N
from d_icd_diagnoses d
inner join 
  (SELECT icd9_code, count(distinct subject_id) as N
  FROM mimiciiiv13.DIAGNOSES_ICD
  where ICD9_code in ('99591', '99592')
  group by icd9_code) j on d.icd9_code = j.icd9_code
where d.ICD9_code in ('99591', '99592')
;
```
###### Count adminissions with sepsis `1,271` or severe sepsis `3,912`
```SQL
select long_title as icd9_name, j.icd9_code, N
from d_icd_diagnoses d
inner join 
(SELECT icd9_code, count(distinct hadm_id) as N
FROM mimiciiiv13.DIAGNOSES_ICD
where ICD9_code in ('99591', '99592')
group by icd9_code) j on d.icd9_code = j.icd9_code
where d.ICD9_code in ('99591', '99592')
;
```

#### Sepsis patients table

##### Considerations
How to calculate age, facts:
  * MIMIC III tutorial propose to calculate age using the first admission time. Many patients have many ICU admissions ,in some cases separated by years.
  * `INTIME` from table `icustays` provides the date and time the patient was transferred into the ICU. 
  * `admittime` from table `admissions` provides the date and time the patient was admitted to the hospital.
  * In some cases the `admittime` preceeds `intime` in othe cases not.
  * Patient >89 get calculate age of hundreds (anonymization tech).

**Conclusion:** calculate the age each time a patient is transferred to ICU

###### Difference between DATETIME and TIMESTAMP
`TIMESTAMP` range is between `'1970-01-01 00:00:01' UTC to '2038-01-09 03:14:07' UTC`, not compatible with mimic III date ranges, i.e., `2164-10-23 21:10:15`.

`DATETIME` range is between `1000-01-01 00:00:00' to '9999-12-31 23:59:59`.

###### Create table with patients with sepsis or severe sepsis, admission information and age at the admission
```SQL
# Required to create a table with invalid dates, for example 0
SET SQL_MODE='ALLOW_INVALID_DATES';
DROP TABLE IF EXISTS sepsis_patients;

# Used datetime instead of timestamp because the first offers a bigger range.
create table sepsis_patients (
	hadm_id int,
	intime DATETIME(0),
	outtime DATETIME(0),
	icd9_code VARCHAR(10),
	SUBJECT_ID int,
	dob DATETIME(0),
	age_admission_icu smallint
)

# Insert data
;
insert into sepsis_patients
select * 
from
(
select j.hadm_id, a.intime, 
	a.outtime, icd9_code, 
	j.subject_id, t.dob, 
	timestampdiff(YEAR, t.dob, a.intime) as age_admission_icu
from icustays a
right join
	(SELECT distinct icd9_code,  hadm_id, subject_id
	FROM mimiciiiv13.DIAGNOSES_ICD
	where ICD9_code in ('99591', '99592')
	) j on a.hadm_id = j.hadm_id
left join 
	(select subject_id, dob
	from patients 
	where subject_id in
		(SELECT distinct subject_id
			FROM mimiciiiv13.DIAGNOSES_ICD
			where ICD9_code in ('99591', '99592')
		) 
	) t on a.subject_id = t.subject_id

where timestampdiff(YEAR, t.dob, a.intime) >= 18
) temp
;

# Index creation
alter table sepsis_patients
	add index sepsis_patients_idx01 (subject_id, hadm_id),
	add index sepsis_patients_idx02 (intime)

;
```

#### Drugs to suspect infection
Only consider IV or IM routes for Anti-infective agents because represents the severity of the suspected infection
```SQL
select distinct GSN, drug 
from PRESCRIPTIONS
where route like 'IV' 
or route like 'IM';"
```

#### Create table with Anti-infective agents
Merge with redbook and select group 'Anti-infective agents'





## References 
  * Bone RC, Balk RA, Cerra FB, Dellinger RP, Fein AM, Knaus WA, Schein RM, Sibbald WJ. Definitions for sepsis and organ failure and guidelines for the use of innovative therapies in sepsis. The ACCP/SCCM Consensus Conference Committee. American College of Chest Physicians/Society of Critical Care Medicine.Chest. 1992 Jun;101(6):1644-55.
http://journal.chestnet.org/article/S0012-3692(16)38415-X/fulltext
  * AHIMA.
  http://library.ahima.org/doc?oid=70222#.WgnxxLA-dTY
  * The DATE, DATETIME, and TIMESTAMP Types.
  https://dev.mysql.com/doc/refman/5.7/en/datetime.html
