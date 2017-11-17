### Determination of biological variables
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
