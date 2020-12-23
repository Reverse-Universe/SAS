### MERGE Function
**Joins observation from two or more SAS datasets into a single observation.**

**Encapsulates the MERGE Satement of SAS's DATA step into one macro function.**

#### Syntax
**%MERGE**_(dataset1, dataset2, feature(s), how, data_merged)_

#### Arguments
***how = left | right | inner | outer***

Type of merge to be performed.
+ left: left join
+ right: right join
+ inner: inner join
+ outer: outer join

#### SAS MACRO Code
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
