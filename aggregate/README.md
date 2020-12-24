# How to AGGREGATE number of visits and number of patients at the same time?

## number of visits
For outpatient clinics,  since outpatients have to go to register before going to the doctors and seeking for consultations, each time a patient goes to outpatient clinics and foots the bills, he will receive two bills — one for registration, and the other for expenditures on medical treatment (in some situations, expenditure bill could be divided into small ones). The hospitals then document each bills for an online database, and use a specific variable (called `reg_or_medi`) to distinguish whether a record in database refers to the bill for registration or medical treatment. Specifically, '1' for registration and '2' for medical treatment.

Therefore, we can create a new variable called `number_of_visits`. Since each time a patient goes to see the doctor, he will only receive one registraion bill, so we could let the `number_of_visits` variable marked by 1.

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

## number of patients
But where it comes to 'number of patients', the problem may be a little bit trickier. We want to know how many patients have received health care at certains times(each year, each month, etc.). Remarkably, one person may have gone to the hospital for several times in a period of time. In this situation ,we regard it as 'one person each period', no matter how many times he have received health care.
Given that every person has been given a unique ID number，which is associated with his own medical insurance card. We can use the variable called `ID_number` to crunch the number of patients who have received the health care.

### Calculating the number of patients
In most cases, we want to know the total number of patients in different groups, for example, hospitals of different level. According to China's three-level health system, each hospital is designated the level of medical service they offer. 
|Level               |Description                    |
|--------------------|-------------------------------|
|First-level hospital|<ul><li>community hospitals</li><li>community healthcare centers</li><li>rural healthcare clinics</li></ul>|
|Second-level hospital|<ul><li>county or district hospitals</li></ul>|
|Third-level hospital|<ul><li>topmost hospitals<ul><li>highly specialized staff</li><li>most-advanced equipment</li><li>well-funded</li><li>most are public hospitals</li></ul></li></ul>|

One of the most conventional approach to count the number of patients from different groups（hospital level） is aggregating `number_of_visits` by different patients and different hospital level at the same time.

```sas
proc means data=dataset nway noprint;
	class ID_number hos_level;
	var number_of_visits;
	output out=result1 sum=;
run;
```
each row represents that how may times has one person visited hospitals of different levels. Then add a new column called `number_of_patients` = 1, and sum up this column according to the value of `hos_level`;
```sas
data result2;
	set result1;
	number_of_patients = 1;
run;

proc means data=result2 nway noprint;
	class hos_level;
	var number_of_patients;
	output out=result3 sum=;
run;
```
### Cleaning the data
However, there exist a fatal problem with the number-of-patient caluculation above. **Remember, never to hope that the raw data in the database is clean enough.** Sometimes we may lose the record for regisration, what's worse, this registration record (`reg_or_medi`=1) may be the only record in the database during a period of time. That means we will miss that patient during aggregation.

Another commonplace is that the record for registration is out of boundary of the period of time during which we would like to calculating the number of patients. For example, the registration bills was recorded at the end of year 2019, however, the medical treatment bills was recorded at the beginning of the year 2020. While calcuating the number of patients in 2020, we miss that patient.

When this happens, a better job would be recognizing that this patient had receive medical care both in the year of 2019 and 2020. If we comply with this principle, how could we handle the problem more properly? 
### Money-related variables
The key to the problem is that both bills(registration and medical treatment) involve expense. The patient could be viewed as having gone to the hospital as long as he had paid for the bill, no matter what kind of that.

Firstly, sum up the total_expense of each patient in each group(`hos_level`). Secondly, while taking the bills of zero-expense (`total_expense` = 0) into consideration, it would be better to drop this kind of bills. Thirdly, add a new column called `number_of_patients` = 1. Finally, aggregate this column by `hos_level`.
```sas
proc means data=dataset nway noprint;
	class ID_number hos_level;
	var total_expense;
	output out=result1 sum=;
run;

data result1;
	set result1;
	where total_expense > 0;
	number_of_patients = 1;
run;

proc means data=result1 nway noprint;
	class hos_level;
	var number_of_patients;
	output out=result2 sum=;
run;
```

## Count the number of patients separately
Here's the variable dictionary for the raw data:
+ **Basic variables**
	+ ID_number
	+ Date
	+ Hospital Code
	+ Hospital Level
	+ Types of Patient <sup>1</sup>
+ **Money related variables**
	+  Total Expense
	+  Uncovered Charges
	+  Coverered Charges
	+  Deductible
	+  Coinsurance
	+  Personal Account Payment
	+  Pooling Acount Payment
+ **Variables for expenses in different categories**
	+ Registration Fee
	+ Diagnosis Fee
	+ Treatment Fee
	+ Surgical Supplies Fee
	+ Hospitalization Fee
	+ Nursing Fee
	+ Examination Fee
	+ Lab Fee
	+ Medical Imaging Fee
	+ X-ray Fee
	+ Blood Transfusion Fee
	+ Oxygen Fee
	+ Medication Fee
	+ Chinese patent medicine Fee
	+ Herbs Fee
	+ Other Fee

<sup> 1. There are five type of patients: outpatient&emergency, inpatient&ICU, critial illness, pharmacy, internal hospital. Different types of patients corresponds to different medical insurance policy (like different deductible and reimbursement proportion). ICU has been excluded from emergency and merged to inpatient, since both ICU patinet and inpatient have to pay for the hospital beds. So the emergency does not include expense on ICU.</sup>

<table>
<tr><td></td><td></td><th>Number of Visits</th><th>Number of Patients</th><th>Total Expense</th></tr>
<tr><th rowspan="3">hospital Level</th><td>Third-level</td><td></td><td></td><td></td></tr>
<tr><td>Second-level</td><td></td><td></td><td></td></tr>
<tr><td>First-level</td><td></td><td></td><td></td></tr>
<tr><th rowspan="2">Community Hospital</th><td>Yes</td><td></td><td></td><td></td></tr>
<tr><td>No</td><td></td><td></td><td></td></tr>
</table>