# Sepsis summary
## Definition of sepsis

Executive Summary

Sepsis is a life-threatening complication related to infection. It occurs when the body induces a systemic inflammatory response (SIRS) that can damage multiple organ systems. Sepsis is a two-part condition (i.e. SIRS and the presence of an infection). Our queries created an algorithm that found patients with sepsis according to the biological parameters as opposed to using ICD9 codes. The accuracy of the algorithm was 97.4. We found that 0.2% of patients diagnosed with sepsis did not have the SIRS. We attributed this to possible hospital transfers where the patient had SIRS and possibly sepsis prior to entering the ICU. The remaining 2.4% is likely due to our discretionary time window. With minor changes, this algorithm could realistically be used to extract sepsis patient information for a more in-depth analysis. One of our group member, Mario is a physician, and had previously worked with MIMICII. Due to this group dynamic characteristic, we wanted to do an advanced query that has potential clinical application. 

Clinical Terminology:
* Hyperthermia – “fever”, elevated temperature
* Hypothermia – reduced body temperature
* Leukocytosis – High white blood cell count
* Leukopenia – Low white blood cell count
* Bands – A premature white blood cell
* Tachycardia – High heart rate
* Hyperventilation – excessive breathing
* PaCO2 Blood measurement of carbon dioxide. Chemical measurement of breathing
* IM - Intramuscular - Drug delivery method that goes directly into the muscle
* IV - Intravenous - Drug delivery method that goes directly into the blood stream. 


Adult patient with a suspected source of infection

#### Systemic inflammatory response syndrome (SIRS)
SIRS is clinically defined as having 2 or more of the following
  * Fever > 100.4 F or hypothermia < 98.6 F
  * Leukocytosis > 12,000 cells/mm, Leukopenia < 4,000 cells/mm or >10% bands
  * Tachycarida > 90
  * Hyperventilation > 20 breaths per minute or PaCO2 < 32 mmHg

### Sepsis codes
| Code        | Name of the code | Requires             |
| ------------|------------------| ---------------------|
| 995.90      | Unspecified SIRS | Underlying condition |
| 995.91      | Sepsis (SIRS due to infectious process without organ dysfunction)           |  Underlying condition, then sepsis code  |
| 995.92      | Severe sepsis (SIRS due to infectious process with organ dysfunction)    |  Underlying condition, severe sepsis code, then organ failure  |


## Create required tables

### Create sepsis_patients table
```SQL
DROP TABLE IF EXISTS sepsis_patients;

# Used datetime instead of timestamp because the first offers a range is consistent with the patient annonymization process.
create table sepsis_patients (
	id_sepsis_admission int PRIMARY KEY AUTO_INCREMENT,
	hadm_id int,
	intime DATETIME(0),
	outtime DATETIME(0),
	icd9_code VARCHAR(10),
	SUBJECT_ID int,
	dob DATETIME(0),
	age_admission_icu smallint
)

# Table J finds patients with sepsis or severe sepsis and the admission ID
# Table T finds patients with sepsise or severe sepsis with their DOB
# Table A is the icustays table, contaning patient information, we derived the length of stay from here
# These tables are joined based off of their admission ID and is filtered to people older than 18
# Insert data
;
insert into sepsis_patients
select NULL, temp.* 
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
We filtered antibiotic use by the two delivery rountes that are standard for sepis patients.
```SQL
select distinct NDC, drug 
from PRESCRIPTIONS
where route like 'IV' 
or route like 'IM';
```

### Create anti infective agents table
Table contains a comprehensive list of all antiobiotics used whiich enabled us to more quickly find patients that received antiobiotic therapy.
This table was imported into the database separate from MIMIC
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

### Create abnormal clinical values table 
**Warning** expensive query, took `6.3` minutes i7 MacBook Pro 15-inch 2017
We created a table based off of the biometric values related to SIRS. To reduce the size 
of the table, we filted to ensure only abnormal values were included.
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
	select hadm_id,  'CO2' as category,itemid, charttime, valuenum
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
**Warning** expensive query, took days for i7 MacBook Pro 15-inch 2017
The Abnorm_clin_val table had 1.2 million rows. This query requried a line by line analysis of that table
```SQL
DROP TABLE IF EXISTS sirs;
create table sirs (
	sirs_id int primary key auto_increment,
	hadm_id int,
	starttime DATETIME(0),
	endtime datetime(0)
)

# Create time interval for SIRS
# obtain id of the next chartevent (the max charttime) that ocurred in the next hour and is not of the same category
# The query is searching for values that are within 1 hour of eachother and returning the most recent timestamp
# Recent time stamp is important for optimizing the window of comparison for antiobiotic administration.

# Insert values
insert into sirs
select NULL, x.*
from (
select start_info.hadm_id, 
	start_info.charttime as starttime, 
	end_info.charttime as endtime
from abnorm_clin_val end_info
right join ( 
		select ab.* , 

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
where start_info.id_end is not null
) x
# Some charevents return NULL because the abnormal values were not recoreded within an hr of eachother.

;

# Index creation
alter table sirs
	add index sirs_idx01(starttime),
	add index sirs_idx02(endtime),
	add index sirs_idx03(hadm_id)
;
```

#### Alternative SIRS table, load from CSV file
Adding this table enabled us to run different queries on the data we dervied from the SIRS query
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
# Index creation
alter table sirs
	add index sirs_idx01(starttime),
	add index sirs_idx02(endtime),
	add index sirs_idx03(hadm_id)
;
```
### Create intervention table
**Warning** query took `2.8` minutes i7 MacBook Pro 15-inch 2017. `61,466` rows.
Finding SIRS patients who were administered antibiotics.
```SQL
DROP TABLE IF EXISTS atb_interventions;
create table atb_interventions (
	atb_int_id int primary key auto_increment,
	hadm_id int,
	drug varchar(50),
	startdate datetime(0),
	enddate datetime(0)
)
;
insert into atb_interventions
select NULL, j.*
from
(
select hadm_id, drug, startdate, enddate
from prescriptions
# Retrieve only patients with sepsis
where hadm_id in (
	select hadm_id
	from sepsis_patients
	)
# Retrieve only anti infective drugs
and ndc in (
	select distinct ndc
	from Anti_infective_drugs)
order by startdate
) j
;
# Index creation
alter table atb_interventions
	add index atb_interventions_val_idx01 (hadm_id),
	add index atb_interventions_val_idx02 (startdate),
	add index atb_interventions_val_idx03 (enddate)
;
```
### Search for SIRS intervals that are inside an antibiotic interval (Sepsis)
**Warning** Expensive query, took `57` minutes i7 MacBook Pro 15-inch 2017
```SQL
select  sirs.*
from sirs
where  exists (select *  
		from atb_interventions a
		where sirs.starttime >= a.startdate
		and sirs.endtime < a.enddate)
```
* It returns `5,037` unique admissions, `97.4%` of the patients with ICD codes for sepis `5,171`.
* Only `5,048`unique admissions presented SIRS, `97.6%` of the patient with ICD codes for sepsis.
* `92.3%` of Error corresponds to admissions that didn't have a SIRS episode according to our algorithm. We should analyze the admissions to find insights about this error. It is possible that some of the admissions didn't have a SIRS episode during their stay at the ICU. Also, we could expand the 60 minutes window to improve sensitivity for SIRS, this will impact the specifitivity of the algorithm and shoulb be evaluated using AUC.

|         | Admission Sepsis ICD + | Admission Sepsis ICD -            |
| ------------|------------------| ---------------------|
| *Predition +*| 5037 | 0 |
| *Prediction -*|   134       | 0  |
| *Total*     | 5171    |  0  |

* Sensitivity (Recall) = `97.40863%`
* Accuracy = `97.40863%`
