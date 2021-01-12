# Create Index on Protected SAS File
## Create Index on Dataset
If you want to access specific records quickly from a large database, the SAS index would help you and endow you with great advantage over traditional sequential access of data.

Indexes are suggested for three types of processing:
+ WHERE Statement in PROC Steps
+ Match Merging
+ BY Statement Processing in DATA/PROC Steps

There are many ways to create an index. Since we can use SQL in the PROC SQL Procedure in SAS, this can be done by using the CREATE INDEX statement like what we do in other database like MySQL. Here is the general format:
```sas
    PROC SQL;
        CREATE INDEX index-name
        ON dataset-name(varlist);
    QUIT;
```
+ **index-name**: the name of the single index variable for Simple indexes and a
+ programmer-chosen name for Composite indexes.<br>
+ **dataset-name**: the name of the SAS data set that the index will be created for.<br>
+ **varlist**: the list of name of variables on which to create index.

The raw data for each healthcare bills are stored in the SAS file / SAS Data Set `tansaction.sas7bdat`. The `PROC SQL` Procedure will automatically create an index file in the format of '.sas7bndx', with the same file name as the original SAS file, in the same folder.
|Name|Size|
|----|----|
|transaction.sas7bdat|50,000,000 KB|
|transaction.sas7bndx|2,000,000 KB|

## Data Security
Preventing the raw data from being modified or tempered by mistakes, or intentionally, is an increasing concern for security leaders and businesses around the world. It is real a good solution: all the raw data (as SAS files) should be set 'Read-Only', any users cannot modify the information of the raw data.

However, after the SAS file being set 'Read-Only', creating index file on it will incur an error:
<pre>
WIRTE ACCESS TO MEMBER 'library.transaction.DATA' IS DENIED.
</pre>
According to my speculation, creating index on the sas file means building up connections between the `.sas7bdat` file and the `.sas7bndx` file, leading to the modification of the `.sas7bdat` file, which is not 'Read-Only'.

## A better Solution: Creating Index indirectly
The transcation database would be a good example, where different insured patients may need to access the database by `person_ID`.

**Target:** create index for the columns `person_ID`

1. Extract the column `person_ID` from all datasets (one record for one healthcare bill), merge them, and then regard it as a dictionary after adding the `row_number` column to it, the `row_number` start from 1 to the total number of rows of the dataset `dictionary`.

```sas
/* Merge all the dateset from different year and month */
data dictionary; set
	/* Jan, 2001 */
	ym0101.outpatient0101 ym0101.inpatient0101 ym0101.severe_disease0101
	ym0101.ICU0101        ym0101.pharmacy0101  ym0101.internal_hos0101

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
```

2. Since the `dictionary` is not 'Read-Only', we could create index for it.
```sas
proc sql;
	create index person_ID
	on dictionary(person_ID);
quit;

```

3. Get the list of person_ID, we would like to search for the records of these person in the raw dataset by `person_ID`.
 
The person_ID_list
|person_ID|
|---------|
|11111111|
|22222222|
|33333333|
|...|

4. Assgin the list to a MACRO variable, more details are shown below.
```sas
proc sql;
	select quote(trim(persion_ID)) into :person_ID_list
	separated by ','
	from person_ID_list;
quit;
```

5. Utilize the index that we have already create to look for certain records in the dictionary.
```sas
data specific_rownum;
	format row_number best16.;
	set dictionary;
	where person_ID in (&person_ID_list);
run;
```

6. We get 'correct' row numbers, assgin it to a new MACRO variable.
```sas
proc sql;
	select row_number into :row_number
	separated by ','
	from specific_rownum;
quit;
```

7. Use the POINT Option to read the certain observations according to the list of row numbers. 
```sas
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


## More details
### The POINT Options in the SET Statement.
**POINT=_variable_**

`SAS 9.4 Statements:Reference, Fifth Edition` says that:

> specifies a temporary variables whose numeric value determines which observation is read. POINT= causes the SET statement to use random (direct) access to read a SAS data set.

**CAUTION:** The POINT= option must be accompained by a STOP statement, otherwise the DATA step would go into a continuous loop. Because POINT= reads only those observations that are specified in the DO statement, SAS cannot read an end-of-file indicator as it would if the file where being read sequentially.

**Here' an excellent example in the official guide:**<br>
Reading a subset by using Direct Access
*These statements select a subset of 50 observations from the data set DRUGTEST by using the POINT= option to access observations directly by number:*
```sas
data sample;
    do obsnum=1 to 100 by 2;
        set drugtest point=obsnum;
        if _error_ then abort;
        output;
    end;
    stop;
run;
```

However, we will meet with some problems if we merge several datasets with different colunms by rows before using the POINT=. For example, we have two dataset -- test1 and test2.<br>
<table>
<tr><th>test1</th><th>test2</th></tr>
<tr><td>
    <table>
        <tr><th>a</th><th>b</th></tr>
        <tr><td>1</td><td>1</td></tr>
        <tr><td>2</td><td>2</td></tr>
        <tr><td>3</td><td>3</td></tr>
    </table>
    </td><td>
    <table>
        <tr><th>c</th><th>d</th></tr>
        <tr><td>1</td><td>1</td></tr>
        <tr><td>2</td><td>2</td></tr>
        <tr><td>3</td><td>3</td></tr>
    </table>
    </td></tr>
</table>

Then combine these two or more datasets, one after the other, into a single dataset. And use the POINT= option for directly accessing data. The value of point refers to the row number in the dataset after combined. For example, if i=5, the second observation of `test2` will be selected, since there are three observations in the `test1`, and `test1` is followed by `test2`.
```sas
data test; 
    do i=1,3,5; 
    set test1 test2 point=i; 
    output; 
    end; 
    stop; 
run;
```
Let's see what happens to the dataset `test`.
|a|b|c|d|
|-|-|-|-|
|1|1|.|.|
|3|3|.|.|
|3|3|2|2|

**Reference**
1. Michael A. Raithel, Westat, Rockville, MD, [Creating and Exploiting SAS® Indexes](https://support.sas.com/resources/papers/proceedings/proceedings/sugi29/123-29.pdf)
2. Alex Vinokurov, Omnicare Clinical Research, King of Prussia, PA, [Using SAS Indexes with Large Databases](https://www.lexjansen.com/nesug/nesug02/bt/bt014.pdf)
3. Mark, Keintz, [A Faster Index for sorted SAS® Datasets](https://www.lexjansen.com/nesug/nesug08/bb/bb10.pdf)