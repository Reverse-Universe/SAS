
```sas
data dictionary; set
	/* Jan, 2001 */
	ym0101.outpatient0101 ym0101.inpatient0101 ym0101.severe_disease0101
	ym0101.ICU0101        ym0101.pharmacy0101        ym0101.internal_hos0101

	/* Feb, 2001 */
	ym0102.outpatient0102 ym0102.hospitalization0102 ym0102.severe_disease0102
	ym0102.ICU0102        ym0102.pharmacy0102        ym0102.internal_hos0102

	...

	/* Dec, 2020 */
	ym2012.outpatient2012 ym2012.hospitalization2012 ym2012.severe_disease2012
	ym2012.ICU2012        ym2012.pharmacy2012        ym2012.internal_hos2012
	;
	keep person_ID;
run;

data dictionary; 
	format row_number best16.;
	set dictionary;
	row_number + 1;
run;

proc sql;
	create index person_ID
	on dictionary(person_ID);
quit;

```

The person_ID_list
|person_ID|
|---------|
|11111111|
|22222222|
|33333333|
|...|

```sas
proc sql;
	select quote(trim(persion_ID)) into :person_ID_list
	separated by ','
	from person_ID_list;
quit;
```

在dictonary查找特定person_ID对应的row_number
```sas
data specific_rownum;
	format row_number best16.;
	set dictionary;
	where person_ID in (&person_ID_list);
run;
```

row_number结果给到row_number_list
```sas
proc sql;
	select row_number into :row_number
	separated by ','
	from specific_rownum;
quit;
```

再在原始表中通过指针快速查询对应记录
data bills;
	do i=&row_number;
		set
		/* Jan, 2001 */
		ym0101.outpatient0101 ym0101.inpatient0101 ym0101.severe_disease0101
		ym0101.ICU0101        ym0101.pharmacy0101        ym0101.internal_hos0101

		/* Feb, 2001 */
		ym0102.outpatient0102 ym0102.hospitalization0102 ym0102.severe_disease0102
		ym0102.ICU0102        ym0102.pharmacy0102        ym0102.internal_hos0102

		...

		/* Dec, 2020 */
		ym2012.outpatient2012 ym2012.hospitalization2012 ym2012.severe_disease2012
		ym2012.ICU2012        ym2012.pharmacy2012        ym2012.internal_hos2012
		;
		point = i;
		output;
		/* Clean the data in PDV */
		<All Variables that will be used later should be align the value of NULL, according to whether it is charater or numeric>
		variable_1 = '';
		variable_2 = .;
		...
		variable_n ='';
	end;
	stop;
run;
```