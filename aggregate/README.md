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
|Non-level hospital|<ul><li>internal hospital<ul><li>university clinic</li><li>enterprise/factory clinic</li></ul></li><li>nursing home</li><li>hospice</li><li>sanatorium</li></ul>|

*Notes: Actually, there are four levels.*

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
Each row of the dataset 'outpatient_settlement' repesents one bill, which indicates how the total amount of expense being decomposited into smaller ones, and how much money insurer and insured should pay towards this bill.

Here's the variable dictionary for the raw data:
+ **Basic variables**
	+ ID_number
	+ Date
	+ Hospital Code
	+ Hospital Level <sup>1</sup>
	+ Community Hospital <sup>2</sup>
	+ Types of Patient <sup>2</sup>
	+ reg_or_medi <sup>3</sup>
+ **Money related variables**<sup>4</sup>
	+  Total Expense
	+  Uncovered Charges
	+  Coverered Charges
	+  Deductible Payment
	+  Payment Above Deductible
	+  Coinsurance Payment
	+  Cash Payment
	+  Individual Account Payment <sup>5</sup>
	+  Pooling Acount Payment <sup>6</sup>
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

<sup>1. Hospital Level: Third-level, Second-level, First-level, Non-level</sup>

<sup>2. the variables `community_hospital` specifies whether a hospital is a  community one. in most cases, the level of community hospitals is *FIRST*. 1 for community hospital and 0 for non-community hospital</sup>

<sup> 3. There are five type of patients: ① outpatient&emergency ② hospitalization(inpatient)&ICU ③ critial illness ④ pharmacy ⑤ internal hospital. Different types of patients correspond to different medical insurance policy (like different deductible and reimbursement proportion). ICU has been excluded from emergency and merged to inpatient, since both ICU patinets and inpatients have to pay for the hospital beds. The emergency does not include expense on ICU.</sup>

<sup>4. `reg_or_medi`: whether a record in database refers to the bill for registration or medical treatment </sup>

<sup> 5. The relationships among these money-related variables are shown below:  </sup>

<sup>&ensp;&ensp;*Total Expense = Uncovered Charges + Covered Charges*</sup>

<sup>&ensp;&ensp;*Covered Charges = Deductible Payment + Payment Above Deductible*</sup>

<sup>&ensp;&ensp;*Payment Above Deductible = Coinsurance Payment + Pooling Account Payment*</sup>

<sup>&ensp;&ensp;*Coinsurance Payment = Individual Account Payment + Cash Payment*</sup>

<sup>6. The insurance fund is divided into two parts: individual account and pooled fund. Patients could pay the deductible and coinsurance by the money on his own individual account</sup>

<sup>7. Pooling Account Payment refers to the expense paid by the insurer, which is the pooled fund.  </sup>

### Task and Traps
**Task:** Aggregate and summarize the outpatient settlement data and output the result in the form of the table below:

<table>
<tr><td></td><td></td><th>Number of Visits</th><th>Number of Patients</th><th>Total Expense</th></tr>
<tr><th rowspan="4">hospital Level</th><td>Third-level</td><td></td><td></td><td></td></tr>
<tr><td>Second-level</td><td></td><td></td><td></td></tr>
<tr><td>First-level</td><td></td><td></td><td></td></tr>
<tr><td>Non-level</td><td></td><td></td><td></td></tr>
<tr><th rowspan="2">Community Hospital</th><td>Yes</td><td></td><td></td><td></td></tr>
<tr><td>No</td><td></td><td></td><td></td></tr>
</table>

**Trap**: Since most community hospitals are also the ones of first level. Is it possible that the number of patient going to the non-community hospital approximately equals to **the sum of** number of patient going to the hospials of second and third level?

Sadly, we can't do this. Supposing that one patient had gone to three hospitals of different levels, each level's `number_of_patient` will increase by one. However, non-community's `number_of_patient` will also increase by one, not two. 

But there is good news: the `number_of_visits` is free of this restriction and could be added together in any way you like.

**TIP: You have to aggregate the number of patients by `hospital_level` and `community_hosptial` separately**

```sas
/* aggregate the number of patients by hospital level */
proc means data=raw_data nway noprint;
	class ID_number hos_level;
	var total_expense;
	output out=id_hos_level sum=;
run;

data id_hos_level;
	set id_hos_level;
	where total_expense > 0;
	number_of_patients = 1;
run;

proc means data=id_hos_level nway noprint;
	class hos_level;
	var number_of_patients;
	output out=hos_level_num_patient sum=;
run;

/* aggregate the number of patients by whether community hospital */
proc means data=raw_data nway noprint;
	class ID_number community_hos;
	var total_expense;
	output out=id_com_hos sum=;
run;

data id_com_hos;
	set id_com_hos;
	where total_expense > 0;
	number_of_patients = 1;
run;

proc means data=id_com_hos nway noprint;
	class community_hos;
	var number_of_patients;
	output out=com_num_patient sum=;
run;
```

## A more complicated case
WHAT IF we want to get the result like this:
<table>
<tr><td></td><td></td><td></td><th colspan='8'>Outpaitent&Emergency</th><th colspan='8'>hospitalization(inpatient)&ICU</th><th>...</th></tr>
<tr><td></td><td></td><td></td><th>Number of Visits</th><th>Number of Patients</th><th>Total Expense</th><th>Uncovered Charges</th><th>...</th><th>Registration Fee</th><th>Diagnosis Fee</th><th>...</th><th>Number of Visits</th><th>Number of Patients</th><th>Total Expense</th><th>Uncovered Charges</th><th>...</th><th>Registration Fee</th><th>Diagnosis Fee</th><th>...</th><th>...</th></tr>

<tr><th rowspan='6'>Year 2010</th><th rowspan='4'>Hospital Level</th><th>Third-level</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><th>Second-level</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><th>First-level</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><th>Non-level</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><th rowspan='2'>Community Hospital</th><th>Yes = 1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><th>No = 0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>

<tr><th rowspan='6'>Year 2011</th><th rowspan='4'>Hospital Level</th><th>Third-level</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><th>Second-level</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><th>First-level</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><th>Non-level</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>

<tr><th rowspan='2'>Community Hospital</th><th>Yes = 1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><th>No = 0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>

<tr><th>...</th><th>...</th><th>...</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>

<tr><th rowspan='6'>Year 2020</th><th rowspan='4'>Hospital Level</th><th>Third-level</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><th>Second-level</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><th>First-level</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><th>Non-level</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>

<tr><th rowspan='2'>Community Hospital</th><th>Yes = 1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
<tr><th>No = 0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>


Two more categorical variables are added to the output table: `YEAR` as the top row header, and `Type of Patients` as the top column header.

`YEAR` includes 2010 to 2020.

`Type of Patients` includes:
+ Outpaitent&Emergency
+ Hospitalization(inpatient)&ICU
+ CriticalIllness
+ Pharmacy
+ InternalHospital

*Notes: Hopitalization and ICU have been put in the same category because both include hospital bed, which Outpatient&Emergency doesn't contains.*

### Begin with a simple problem
IF the `Number of Patients` are omitted, the problem will become much easier, and by that I mean, we will only need **PROC MEANS** and **PROC TABLULATE** procedures: 
```sas
proc means data=raw_data nway noprint;
	class year hos_level community_hos type_of_patients;
	var number_of_visits
		total_expense uncovered_charges <other money related variables>
		registration_fee diagnosis_fee <other variables for expenses in different categories>;
	output out=result1 sum=;
run;

proc tabulate data=result1;
	class year hos_level community_hos type_of_patients;
	var number_of_visits
		total_expense uncovered_charges <other money related variables>
		registration_fee diagnosis_fee <other variables for expenses in different categories>;
	/* the CLASS and VAR statement are just the same as that of PROC MEANS */
	table year=''*(hos_level='' community_hos=''), type_of_patients=''*sum=''*f=20.0*(number_of_visits
		total_expense uncovered_charges <other money related variables>
		registration_fee diagnosis_fee <other variables for expenses in different categories>) 
		/ style={width=5000};
run;
```

### Add 'number_of_patients'
Although the `number_of_visits` and `number_of_patients` cannot be aggregated at the same time (in the same **PROC MEANS** procedure), `year` and `type_of_patients` could be in the same CLASS statement.

***KEY***: The aggregation of `number_of_patients` is DANGEROUS and should be handled meticulously if the categorical variables are paralleled in the output table. A nested structure is more welcomed. For example, `hos_level` and `community_hos` are paralleled in the table, so the aggregation of `number_of_patients` should also be paralleled. The `year` and `community_hos` are not paralleled, in details, the `community_hos` is nested within the `year`, so we don't have to worry about the problem of one-patient-in-two-groups, in which one patient has been to the community hosital with first_level. 

***TIP***: Different years' `number_of_patients` do not tangle with each other, that's what we hope.

While aggregating the `number_of_patients` by different years and patients' types, just put 'year type_of_patients' after the CLASS statement in PROC MEANS procedures. Keep in mind that `number_of_patients` in different hospital levels and in communtiy hospitals should be counted separately. 
```sas
/* aggregate the number of patients by hospital level of different year and type_of_patients */
proc means data=raw_data nway noprint;
	class ID_number hos_level year type_of_patients;
	var total_expense;
	output out=id_hos_level sum=;
run;

data id_hos_level;
	set id_hos_level;
	where total_expense > 0;
	number_of_patients = 1;
run;

proc means data=id_hos_level nway noprint;
	class hos_level year type_of_patients;
	var number_of_patients;
	output out=hos_level_num_patient sum=;
run;

/* aggregate the number of patients by whether community hospital of different year and type_of_patients*/
proc means data=raw_data nway noprint;
	class ID_number community_hos year type_of_patients;
	var total_expense;
	output out=id_com_hos sum=;
run;

data id_com_hos;
	set id_com_hos;
	where total_expense > 0;
	number_of_patients = 1;
run;

proc means data=id_com_hos nway noprint;
	class community_hos year type_of_patients;
	var number_of_patients;
	output out=com_num_patient sum=;
run;
```
The structure of dataset `hos_level_num_patient.sas` and all the possible values for each variables.
|year|type_of_patients|hos_level|number_of_patients|
|----|----------------|---------|------------------|
|2010,2011,...,2020|5 Types|Third, Second, First, Non|  |

Total number of rows: 11 × 5 × 4 = 220

The structure of dataset `com_num_patient.sas` and all the possible values for each variables.
|year|type_of_patients|community_hos|number_of_patients|
|----|----------------|-------------|------------------|
|2010,2011,..,2020|5 Types|0,1| |

*Notes: 1 for community hospital and 0 for non-community hospital*

Total number of rows: 11 × 5 × 2 = 110
### MERGE into ONE
As mentioned above, we could output the result table without `number_of_patients` directly under the help of PROC MEANS and PROC TABULATE procedures (we have output it as `result1.sas` above). However, the structures of result table `hos_level_num_patient` and `com_num_patient` are quite different from that. That means these three dataset cannot be merged directly.
## The structure of result1.sas
Simplification of `result1.sas`: hide the `year` and `type_of_patients` temporarily, this could help us understand the tricks for the problem.
|hos_level|community_hos|number_of_visits|...|
|---------|-------------|----------------|---|
|Non-level|0|||
|First-level|0|||
|First-level|1|||
|Second-level|0|||
|Third-level|0|||



