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
