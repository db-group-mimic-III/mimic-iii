# Sepsis summary
## Create required tables

### Create sepsis_patients table
```SQL
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

### Create anti infective agents table 
```SQL
alter table sepsis_patients
	add index sepsis_patients_idx01 (subject_id, hadm_id),
	add index sepsis_patients_idx02 (intime)

;

### Anti infective agents table

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

### Create abnormam clinical values table 
**Warning** expensive query, took `6.3` minutes i7 MacBook Pro 15-inch 2017
``` SQL
DROP TABLE IF EXISTS abnorm_clin_val;

create table abnorm_clin_val (
	id_ab_clin int primary key auto_increment,
	hadm_id int,
	category varchar(15),
	itemid int,
	charttime DATETIME(0),
	valuenum double precision		
)
;

# CO2
insert into abnorm_clin_val
select NULL, j.* from (

	select hadm_id, 'CO2' as  category, itemid, charttime, valuenum
	from labevents
	where hadm_id in
		(
		select hadm_id
		from sepsis_patients #where subject_id = 10188
		)
	and itemid in (50818,  50804)
	and valuenum < 32
	) j
;



# WBC
insert into abnorm_clin_val
select NULL, j.* from (
	select hadm_id, 'WBC' as category,itemid, charttime, valuenum
	from labevents
	where hadm_id in
		(
		select hadm_id
		from sepsis_patients #where subject_id = 10188
		)
	and  itemid in  (51300, 51301)
	and (valuenum < 4 or valuenum > 12)
	) j
;

# Bands
insert into abnorm_clin_val
select NULL, j.* from (
	select hadm_id, 'Bands' as category,itemid, charttime, valuenum
	from labevents
	where hadm_id in
		(
		select hadm_id
		from sepsis_patients #where subject_id = 10188
		)
	and  itemid in  (51144)
	and valuenum >10
	) j
;


# Heart
insert into abnorm_clin_val
select NULL, j.* from (
	select  hadm_id, 'Heart rate' as category,itemid, charttime, valuenum
	from chartevents
	where hadm_id in
		(
		select hadm_id
		from sepsis_patients #where subject_id = 10188
		)
	and  itemid in  (211, 220045)
	and valuenum >90
	) j
;


# Temp F
insert into abnorm_clin_val
select NULL, j.* from (
	select hadm_id, 'Temp F' as category,itemid, charttime, valuenum
	from chartevents
	where hadm_id in
		(
		select hadm_id
		from sepsis_patients #where subject_id = 10188
		)
	and  itemid in  (679,678)
	and (valuenum >100.4 or valuenum < 98.6)
	) j
;

# Resp rate 
insert into abnorm_clin_val
select NULL, j.* from (
	select hadm_id,  'Resp rate' as category,itemid, charttime, valuenum
	from chartevents
	where hadm_id in
		(
		select hadm_id
		from sepsis_patients #where subject_id = 10188
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
### Create SIRS table
**Warning** expensive quey, took `xx` days i7 MacBook Pro 15-inch 2017
```SQL
DROP TABLE IF EXISTS sirs;
create table sirs (
	sirs_id int primary key auto_increment,
	hadm_id int,
	starttime DATETIME(0),
	endtime datetime(0)
)


# Inser values
select start_info.hadm_id, 
	start_info.charttime as starttime, 
	end_info.charttime as endtime
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
;
# Index creation
alter table sirs
	add index sirs_idx01(starttime),
	add index sirs_idx02(endtime),
	add index sirs_idx03(hadm_id)
;
```

#### Alternative SIRS table, load from CSV file
```SQL
DROP TABLE IF EXISTS sirs;
create table sirs (
	sirs_id int primary key auto_increment,
	hadm_id int,
	starttime DATETIME(0),
	endtime datetime(0)
)
;

LOAD DATA LOCAL INFILE 'sirs.csv' INTO TABLE sirs
   FIELDS TERMINATED BY ',' ESCAPED BY '\\' OPTIONALLY ENCLOSED BY '"'
   LINES TERMINATED BY '\n'
   IGNORE 1 LINES
   (@hadm_id,@starttime, @endtime)
 SET
   hadm_id = @hadm_id,
   starttime = @starttime,
   endtime = @endtime
   ;
```
