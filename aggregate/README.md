## How to AGGREGATE number of visits and number of people at the same time?

### number of visits
For outpatient clinics,  since outpatients have to go to register before going to the doctors and seeking for consultations, each time a patient goes to outpatient clinics and foots the bills, he will receive two bills — one for registration, and the other for expenditures on medical treatment (in some situations, expenditure bill could be divided into small ones). The hospitals then document each bills for an online database, and use a specific variable (called `reg_or_medi`) to distinguish whether a record in database refers to the bill for registration or medical treatment. Specifically, '1' for registration and '2' for medical treatment.

Therefore, we can create a new variable called `number_of_visits`, each time a patient goes to the hospital, he will only receive one registraion bills, so we could let the `number_of_visits` variable marked by 1.

<pre>
if reg_or_medi='1' then number_of_visits=1;
else number_of_visits=0;
</pre>

The variable `number_of_visits` may do some help and simplify the code if we aggreate it and other variables at the same time:

```sas
proc means data=dataset nway noprint;
	var number_of_visits <other numeric variables>;
	class <categorical variables>;
	output out=result sum=.;
run;
```

### number of people
But where it comes to 'number of people', the problem may be a little bit trickier. We want to know how many people have received health care at certains times(each year, each month, etc.). Remarkably, one person may have gone to the hospital for several times in a period of time. In this situation ,we regard it as 'one person each period', no matter how many times he have received health care.
Given that every person has been given a unique ID number，which is associated with his own medical insurance card. We can use the variable called `ID_number` to crunch the number of people who have received the health care.

#### Clean the data first
In most cases, we want to know the total number of people in different groups, for example, hospitals of different level. According to China's three-level health system, each hospital is designated the level of medical service they offer. 
|Level               |Description                    |
|--------------------|-------------------------------|
|First-level hospital|<ul><li>community hospitals</li><li>community healthcare centers</li><li>rural healthcare clinics</li></ul>|
|Second-level hospital|<ul><li>county or district hospitals</li></ul>|
|Third-level hospital|<ul><li>topmost hospitals<ul><li>highly specialized staff</li><li>most-advanced equipment</li><li>well-funded</li><li>most are public hospitals</li></ul></li></ul>|

One of the most conventional approach to count the number of people from different groups（hospital level） is aggregating number_of_visits by different people and different hospital level at the same time.

```sas
proc means data=dataset nway noprint;
	class ID_number hos_level;
	var number_of_visits;
run;
```
each row represents that how may times has one person visited hospitals of different levels.
