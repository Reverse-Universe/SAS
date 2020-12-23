<table>
<tr> <th>MACRO Function</th> <th>Description</th> <th>Syntax</th> <th>SAS Code</th> <th>Example</th> <th>HowToMemorize</th> </tr>
<tr>
	<td>%MERGE()</td>
	<td>Joins observation from two or more SAS datasets into a single observation.</td>
	<td>%merge(dataset1, dataset2, feature(s), how, data_merged)</td>
	<td>

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

%mend merge;
```

	</td>
	<td>%merge(a,b,ziduan,inner,new_ab)</td>
	<td>pd.merge() in Pandas</td>
</tr>
</table>