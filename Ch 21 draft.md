-- This query extracts data for the tutorial in the clinical data analytics book chapter. Only the first icu stays from adult patients are extracted.
````sql
DROP TABLE IF EXISTS static_data;
create table static_data as
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
# Index creation
alter table static_data
	add index static_data_idx01 (subject_id),
	add index static_data_idx02 (hadm_id),
	add index static_data_idx03 (ICUSTAY_ID)
	
;

-----------------------------
-- BEGIN EXTRACTION OF LABS 
-----------------------------
DROP TABLE IF EXISTS labevents_21;
create table labevents_21 as
(select hadm_id,
        itemid,
        charttime,
        valuenum
 from mimiciiiv13.labevents l
 where itemid in (50912, 50971, 50983, 509802, 50882, 51221, 51300, 50931, 50960, 50893, 50970, 50813)
   and hadm_id in (select hadm_id from static_data) 
   and valuenum is not null
)
;
# Index creation
alter table labevents_21
	add index labevents_21_idx01 (hadm_id),
	add index labevents_21_idx02 (charttime)
;



DROP TABLE IF EXISTS labs_raw;
 create table labs_raw as
select a.*
from labevents_21 a, (
	select hadm_id, itemid , min(charttime) as min_charrtime
	from labevents_21
	group by hadm_id, itemid
									) b
where a.hadm_id = b.hadm_id and a.itemid = b.itemid and a.charttime = b.min_charrtime
;
# Index creation
alter table labs_raw
	add index labs_raw_idx01 (hadm_id),
	add index labs_raw_idx02 (itemid)
;

select * from labs_raw;

select  hadm_id,
			sum(case itemid when 50912 then valuenum else NULL END) as cr,
			sum(case itemid when 50971 then valuenum else NULL END) as k,
			sum(case itemid when 50983 then valuenum else NULL END) as na,
			sum(case itemid when 50902 then valuenum else NULL END) as cl,
			sum(case itemid when 50882 then valuenum else NULL END) as bicarb,
			sum(case itemid when 51221 then valuenum else NULL END) as htc,
			sum(case itemid when 51300 then valuenum else NULL END) as wbc,
			sum(case itemid when 50931 then valuenum else NULL END) as glucose,
			sum(case itemid when 50960 then valuenum else NULL END) as mg,
			sum(case itemid when 50893 then valuenum else NULL END) as ca,
			sum(case itemid when 50970 then valuenum else NULL END) as p,
			sum(case itemid when 50813 then valuenum else NULL END) as lactate
from  labs_raw 
group by hadm_id
;
--select * from labs;
------------------------------
--- END OF EXTRACTION OF LABS
------------------------------

------------------------------------
--- BEGIN EXTRACTION OF VITALS
------------------------------------
DROP TABLE IF EXISTS small_charevents;
create table small_charevents as
select ICUSTAY_ID,
	
        case
         when itemid in (211, 220045) then 'hr'
         when itemid in (52,456, 220052) then 'map'  -- invasive and noninvasive measurements are combined
         when itemid in (51,455, 220050) then 'sbp'  -- invasive and noninvasive measurements are combined
         when itemid in (678, 679, 223761) then 'temp'  -- in Fahrenheit
         when itemid in (646, 226253) then 'spo2'    -- values from chartevents, not labevents
         when itemid in (618, 220210) then 'rr'
        end as type,                
        charttime,
        valuenum
 from mimiciiiv13.chartevents l
 where itemid in (211,51,52,455,456,678,679,646,618,220045,220052,220050,223761,220210 )-- note: dont have spo2 value yet
   and ICUSTAY_ID in (select ICUSTAY_ID from static_data) 
   and valuenum is not null
;
alter table labevents_21
	add index labevents_21_idx01 (subject_id),
	add index labevents_21_idx02 (hadm_id),
	add index labevents_21_idx03 (ICUSTAY_ID),
	add index labevents_21_idx04 (itemid)



DROP TABLE IF EXISTS vitals_raw;
create table vitals_raw as
select a.*
from small_charevents a, (
				select ICUSTAY_ID, type, min(charttime) as min_charttime
				from small_charevents 
				group by ICUSTAY_ID, type
				) b
where a.ICUSTAY_ID = b.ICUSTAY_ID and a.type = b.type and a.charttime = b.min_charttime


alter table labevents_21
	add index vitals_raw_idx01 (subject_id),
	add index vitals_raw_idx02 (hadm_id),
	add index vitals_raw_idx03 (ICUSTAY_ID),
	add index vitals_raw_idx04 (itemid)

------------------------------------
--- END OF EXTRACTION OF VITALS
------------------------------------
# Assemble table

select *
from
(
select ICUSTAY_ID,
			sum(case type when 'hr' then valuenum else NULL END) as 'hr',
			sum(case type when 'map' then valuenum else NULL END) as 'map',
			sum(case type when 'sbp' then valuenum else NULL END) as 'sbp',
			sum(case type when 'temp' then valuenum else NULL END) as 'temp',
			sum(case type when 'spo2' then valuenum else NULL END) as 'spo2',
			sum(case type when 'rr' then valuenum else NULL END) as 'rr'
from vitals_raw
group by ICUSTAY_ID
) z,
(
select  ICUSTAY_ID,
			sum(case itemid when 50912 then valuenum else NULL END) as cr,
			sum(case itemid when 50971 then valuenum else NULL END) as k,
			sum(case itemid when 50983 then valuenum else NULL END) as na,
			sum(case itemid when 50902 then valuenum else NULL END) as cl,
			sum(case itemid when 50882 then valuenum else NULL END) as bicarb,
			sum(case itemid when 51221 then valuenum else NULL END) as htc,
			sum(case itemid when 51300 then valuenum else NULL END) as wbc,
			sum(case itemid when 50931 then valuenum else NULL END) as glucose,
			sum(case itemid when 50960 then valuenum else NULL END) as mg,
			sum(case itemid when 50893 then valuenum else NULL END) as ca,
			sum(case itemid when 50970 then valuenum else NULL END) as p,
			sum(case itemid when 50813 then valuenum else NULL END) as lactate
from  labs_raw 
group by ICUSTAY_ID
) y
where z.ICUSTAY_ID = y.ICUSTAY_ID
