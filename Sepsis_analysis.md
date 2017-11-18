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
	id_ab_clin int PRIMARY KEY AUTO_INCREMENT,
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
or route like 'IM';
```

#### Create table with Anti-infective agents
Merge with redbook and select group 'Anti-infective agents'
``` SQL
DROP TABLE IF EXISTS Anti_infective_drugs;
CREATE TABLE Anti_infective_drugs
(
# Duplicated NDC
	NDC varchar(15) ,
	Drug varchar(50)
  )	
;

LOAD DATA LOCAL INFILE 'anti_infective_drugs.csv' INTO TABLE anti_infective_drugs
   FIELDS TERMINATED BY ',' ESCAPED BY '\\' OPTIONALLY ENCLOSED BY '"'
   LINES TERMINATED BY '\n'
   IGNORE 1 LINES
   (@NDC,@drug)
 SET
   NDC = @NDC,
   Drug = @drug
   ;
```


#### Create table with abnormal clinical values using sepsis_patients
``` SQL
DROP TABLE IF EXISTS abnorm_clin_val;

create table abnorm_clin_val (
	hadm_id int,
	category varchar(15),
	itemid int,
	charttime DATETIME(0),
	valuenum double precision		
)
;

# CO2
insert into abnorm_clin_val
select * from (

	select hadm_id, 'CO2' as  category, itemid, charttime, valuenum
	from labevents
	where hadm_id in
		(
		select hadm_id
		from sepsis_patients where subject_id = 10188
		)
	and itemid in (50818,  50804)
	and valuenum < 32
	) j
;



# WBC
insert into abnorm_clin_val
select * from (
	select hadm_id, 'WBC' as category,itemid, charttime, valuenum
	from labevents
	where hadm_id in
		(
		select hadm_id
		from sepsis_patients where subject_id = 10188
		)
	and  itemid in  (51300, 51301)
	and (valuenum < 4 or valuenum > 12)
	) j
;

# Bands
insert into abnorm_clin_val
select * from (
	select hadm_id, 'Bands' as category,itemid, charttime, valuenum
	from labevents
	where hadm_id in
		(
		select hadm_id
		from sepsis_patients where subject_id = 10188
		)
	and  itemid in  (51144)
	and valuenum >10
	) j
;


# Heart
insert into abnorm_clin_val
select * from (
	select  hadm_id, 'Heart rate' as category,itemid, charttime, valuenum
	from chartevents
	where hadm_id in
		(
		select hadm_id
		from sepsis_patients where subject_id = 10188
		)
	and  itemid in  (211, 220045)
	and valuenum >90
	) j
;


# Temp F
insert into abnorm_clin_val
select * from (
	select hadm_id, 'Temp F' as category,itemid, charttime, valuenum
	from chartevents
	where hadm_id in
		(
		select hadm_id
		from sepsis_patients where subject_id = 10188
		)
	and  itemid in  (679,678)
	and (valuenum >100.4 or valuenum < 98.6)
	) j
;

# Resp rate 
insert into abnorm_clin_val
select * from (
	select hadm_id,  'Resp rate' as category,itemid, charttime, valuenum
	from chartevents
	where hadm_id in
		(
		select hadm_id
		from sepsis_patients where subject_id = 10188
		)
	and  itemid = 618
	and valuenum  > 20 
	) j
;

#  Index creation
alter table abnorm_clin_val
	add index abnorm_clin_val_idx01 (charttime),
	add index abnorm_clin_val_idx02 (category)
;

```

#### Create SIRS intervals
```SQL
select start_info.*, 
	end_info.category, 
	end_info.charttime as end_time, 
	end_info.valuenum, 
	timestampdiff(MINUTE, start_info.charttime, end_info.charttime) as time_diff_min
from abnorm_clin_val end_info
right join ( 
		select ab.* , 
# Create time interval for SIRS
# obtain id of the next chartevent (the max charttime) that ocurred in the next hour and is not of the same category
		(select  id_ab_clin
		from abnorm_clin_val j
		where j.category != ab.category
		and timestampdiff(MINUTE, ab.charttime, j.charttime)  < 60
		and timestampdiff(MINUTE, ab.charttime, j.charttime)  >=0
		order by charttime desc
		limit 1
		) as id_end
		from abnorm_clin_val ab
	) start_info on end_info.id_ab_clin = start_info.id_end
# Some charevents return NULL because you don't have a temporal match
where start_info.id_end is not null
```

## References 
  * Bone RC, Balk RA, Cerra FB, Dellinger RP, Fein AM, Knaus WA, Schein RM, Sibbald WJ. Definitions for sepsis and organ failure and guidelines for the use of innovative therapies in sepsis. The ACCP/SCCM Consensus Conference Committee. American College of Chest Physicians/Society of Critical Care Medicine.Chest. 1992 Jun;101(6):1644-55.
http://journal.chestnet.org/article/S0012-3692(16)38415-X/fulltext
  * AHIMA.
  http://library.ahima.org/doc?oid=70222#.WgnxxLA-dTY
  * The DATE, DATETIME, and TIMESTAMP Types.
  https://dev.mysql.com/doc/refman/5.7/en/datetime.html
