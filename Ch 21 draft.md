-- This query extracts data for the tutorial in the clinical data analytics book chapter. Only the first icu stays from adult patients are extracted.
````sql
create view static_data as
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
select * from static_data;

-----------------------------
-- BEGIN EXTRACTION OF LABS # Code is good!!!
-----------------------------
 create view labevents_21 as
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
select * from labevents_21;



-- ****************** look at code!

 create table labs_raw as
select a.*
from labevents_21 a, (
	select hadm_id, itemid , min(charttime) as min_charrtime
	from labevents_21
	group by hadm_id, itemid
									) b
where a.hadm_id = b.hadm_id and a.itemid = b.itemid and a.charttime = b.min_charrtime


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
, small_chartevents as
(select icustay_id,
        case
         when itemid in (211, 220045) then 'hr'
         when itemid in (52,456, 220052) then 'map'  -- invasive and noninvasive measurements are combined
         when itemid in (51,455, 220050) then 'sbp'  -- invasive and noninvasive measurements are combined
         when itemid in (678, 679, 223761) then 'temp'  -- in Fahrenheit
         when itemid in (646, 226253) then 'spo2'    -- values from chartevents, not labevents
         when itemid in (618, 220210) then 'rr'
        end as type,                
        charttime,
        value1num
 from mimiciiiv13.chartevents l
 where itemid in (211,51,52,455,456,678,679,646,618,220045,220052,220050,223761,220210 )-- note: dont have spo2 value yet
   and icustay_id in (select icustay_id from static_data) 
   and value1num is not null
)
--select * from small_chartevents;

, vitals_raw as
(select distinct icustay_id,        
        type,
        first_value(value1num) over (partition by icustay_id, type order by charttime) as first_value
 from small_chartevents 
)
--select * from vitals_raw;

, vitals as
(select *
 from (select * from vitals_raw)
      pivot
      (sum(round(first_value,1)) as admit
       for type in 
       ('hr' as hr,
        'map' as map,
        'sbp' as sbp,
        'temp' as temp,
        'spo2' as spo2,
        'rr' as rr
       )
      )
)
--select * from vitals;
------------------------------------
--- END OF EXTRACTION OF VITALS
------------------------------------

-- Assemble final data
, final_data as
(select s.*,
        v.hr_admit,
        v.map_admit,
        v.sbp_admit,
        v.temp_admit,
        v.spo2_admit,       
        v.rr_admit,       
        l.cr_admit, 
        l.k_admit,
        l.na_admit,
        l.cl_admit,
        l.bicarb_admit,
        l.hct_admit,
        l.wbc_admit,
        l.glucose_admit,
        l.mg_admit,
        l.ca_admit,
        l.p_admit,
        l.lactate_admit
 from static_data s
 left join vitals v on s.icustay_id=v.icustay_id 
 left join labs l on s.icustay_id=l.icustay_id 
)
select * from final_data order by 1,2,3;
````
