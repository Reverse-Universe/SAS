# MATCH MERGE
**Joins observation from two or more SAS datasets into a single observation.**

**Encapsulates the MERGE Satement of SAS's DATA step into one macro function.**

## Syntax
**%MERGE**_(dataset1, dataset2, feature(s), how, data_merged)_

## Arguments
***how = left | right | inner | outer***

Type of merge to be performed.
+ left: left join
+ right: right join
+ inner: inner join
+ outer: outer join

***features(s)***

While match-merging two datasets, The argument `feature(s)` specifies the variables to merge by. You can assign multiple variables (separated by blanks) to this argument.

## SAS MACRO Code
```sas
%macro merge(dataset1, dataset2, features, how, data_merged);
	%if &how=left %then %do;
		proc sort data=&dataset1; by &features; run;
		proc sort data=&dataset2; by &features; run;
		data &data_merged;
			merge &dataset1(in=in1) &dataset2;
			by &features;
			if in1;
		run;
	%end;

	%if &how=right %then %do;
		proc sort data=&dataset1; by &features; run;
		proc sort data=&dataset2; by &features; run;
		data &data_merged;
			merge &dataset1 &dataset2(in=in1);
			by &features;
			if in1;
		run;
	%end;

	%if &how=inner %then %do;
		proc sort data=&dataset1; by &features; run;
		proc sort data=&dataset2; by &features; run;
		data &data_merged;
			merge &dataset1(in=in1) &dataset2(in=in2);
			by &features;
			if in1 and in2;
		run;
	%end;

	%if &how=outer %then %do;
		proc sort data=&dataset1; by &features; run;
		proc sort data=&dataset2; by &features; run;
		data &data_merged;
			merge &dataset1(in=in1) &dataset2(in=in2);
			by &features;
			if in1 or in2;
		run;
	%end;

%mend merge;
```

## How to Memorize
While in Python, we have:
```python
import pandas as pd
new_df = pd.merge(df1, df2, on = 'label or list', how = 'left')
```
They look quite similar.


# DROP DUPLICATES
**Return dataset with duplicates removed**

**Drop duplicates in the DATA Step, rather than PROC Step**

## Syntax
**%DROP_DUPLICATES**_(dataset, feature(s), keep, new_dataset)_

## Arguments
***dataset***<br>
*The table that contains duplicates*<br>
<br>
***feature(s)***<br>
*The certain columns for identifying duplicates. The name of columns should be separated by BLANKs like 'a b c'*<br>
<br>
***keep***<br>
*Determines whether to keep the first or last occurrence among the duplicates.*<br>
&ensp;&ensp;*__first__ : Drop duplicates except for the first occurrence.*<br>
&ensp;&ensp;*__last__ : Drop duplicates except for the last occurrence.*<br>
<br>
***new_dataset***<br>
*The output table without duplicates, keeping it NULL will drop duplicates in place, which means overwriting the primitve table which contains duplicates*<br>

## SAS MACRO Code
```sas
%macro drop_duplicates(dataset, features, keep, new_dataset);

	%let dot=.;
	%let last_feature = %scan(&features, -1); /* Get the last word of features */
	%put &last_feature; /* Check the value of MACRO variable: last_feature */

	proc sort data = &dataset;
		by &features;
	run;

	/* Check whether the value of new_dataset is NULL */
	%if &new_dataset = %then %do;
		%let output_dataset = &dataset;
	%end;
	%else %do;
		%let output_dataset = &new_dataset;
	%end;

	data &output_dataset;
		set &dataset;
		by &features;

		%if &keep=first %then %do;
			if first&dot.&last_feature;
		%end;
		%else %if &keep=last %then %do;
			if last&dot.&last_feature;
		%end;
	run;

%mend drop_duplicates;
```

## Exmaples
The table `test` holds the data of:

|a|b|c|d|
|-|-|-|-|
|1|2|3|8|
|1|2|4|9|
|5|2|4|9|
|1|2|3|9|

```sas
%drop_duplicates(test, a b c, first, test_new1);
%drop_duplicates(test, a b c, last, test_new2);
%drop_duplicates(test, a b c, last);
```

Then we get the new table `test_new1`:

|a|b|c|d|
|-|-|-|-|
|1|2|3|8|
|1|2|4|9|
|5|2|4|9| 

The last row has been removed, and `test_new2`:

|a|b|c|d|
|-|-|-|-|
|1|2|3|9|
|1|2|4|9|
|5|2|4|9| 

The first row has been removed.

The `test` has been modified to:

|a|b|c|d|
|-|-|-|-|
|1|2|3|9|
|1|2|4|9|
|5|2|4|9| 