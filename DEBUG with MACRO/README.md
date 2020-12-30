# Handling System Error

## Automatic Macro Variable: SYSERR
**SYSERR** is used to detect major system errors. Once SAS reports error in a STEP, a certain code will be assign to **SYSERR**.

|Code|Description|
|----|-----------|
|0|Execution completed successfully and without warning messages.|
|1|Execution was canceled by a user with a RUN CANCEL statement.|
|...|...|

**Warning Codes**
|Code|Description|
|----|-----------|
|108|Problem with one or more BY groups|
|...|...|

**SYSERR Error Codes**
|Code|Description|
|----|-----------|
|1008|General data problem|
|1012|General error condition|
|...|...|

_Notes: IF the value of **SYSERR** is TRUE (>0), there exists warnings or errors._

It is worth mentioning that **SYSERR** automatic macro variable is reset at each step boundary, which means the value of SYSERR will be refreshed at the end of each STEP, so we have to assign the latest value to the global macro variable at the end of each STEP.

## TASK and SOLUTION
The size of raw data in the library is listed as following:

|Name of SAS file|Estimated Size|
|-----------------------------|--------------|
|Year2010.sas|100 GB|
|Year2011.sas|150 GB|
|Year2012.sas|200 GB|
|...|Δsize = 50 GB for each year|
|Year2020.sas|600 GB|

_**Notes:**_


*&ensp;&ensp;Total size of the disks = 5 TB = 5120 GB*


*&ensp;&ensp;Total size of SAS file as raw data = 3850 GB*


*&ensp;&ensp;Remaining diskspace for Work library = 5120 - 3850 = 1270 GB*


>*The Work library is a special-purpose SAS library that contains temporary files, including certain types of utility files that are created by SAS as part of processing the current SAS session or job.*

**TASK:** aggregate all the numeric variables by all the categorical variables.
```sas
/* merge these sas file into one dataset first */
data overall; set
    lib.year2010 lib.year2011 lib.year2012 lib.year2013 lib.year2014
    lib.year2015 lib.year2016 lib.year2017 lib.year2018 lib.year2019 lib.year2020
    ;
    <WHERE Statement>;
    <IF Statement>;
run;

proc means data=overall nway noprint;
    class <all categorical variables>;
    var <all numeric variables>;
    output out=result sum=;
run;
```
Unfortunately, the remaining disk space is not enough for SAS to create dataset `overall`, since this DATA Step would try to merge all the data into a two one at once. Finally, the SAS pops out a window, warning that the disk is **OUT OF RESOURCE**.

**Solution:** Read one year's data each time, aggregate it before reading next year's data. DON'T FORGET to add a new variable called `year` after aggregation, otherwise we can not tell the difference between data of different year. **DELETE raw data in the Work library after aggregated into new dataset.**
```sas
data y2010; set
    lib.year2010;
    <WHERE Statement>;
    <IF Statement>;
run;

<Other operation on y2010.sas, like inner join>

proc means data=y2010 nway noprint;
    class <all categorical variables>;
    var <all numeric variables>;
    output out=result2010 sum=;
run;

data result2010;
    set result2010;
    year = 2010;
run;

/* Now the raw datay 'y2010' are redundant, we don't need it anymore, delete it from the disk to save diskspace */

proc datasets lib=work;
    delete y2010;
run;

/* code for other years, or you can write SAS MACRO function */
<SAS Code>
/* code for other years, or you can write SAS MACRO function */

/* Merge them into final result */
data result;set
    y2010 y2011 y2012 y2013 y2014 y2015 y2016 y2017 y2018 y2019 y2020
    ;
run;
```
**CAUTION**: DON'T DELETE the dataset in `lib`, where we store the raw data. Delete `work.y2010`, not `lib.year2010`.

### Problem
If there are some syntax errors among the SAS code. For example, we run the overall program (from year 2010 to 2020), and hope all that works. However, the **PROC MEANS** procedure on dataset `y2015.sas` may have some errors, the dataset `result2015.sas` will not be created, the SAS still delete the raw data `y2015.sas` because the SAS will not stop upon the fiest warning or error, except that you turn on the system option `ERRORABEND`:
```sas
OPTIONS ERRORABEND;
```
>Use the ERRORABEND system option with SAS production programs, which presumably should not encounter any errors. If errors are encountered and ERRORABEND is in effect, SAS brings the errors to your attention immediately by terminating. ERRORABEND does not affect how SAS handles notes such as invalid data messages.

You click on the run botton, and check the result on the next morning, finding out that *PROC MEANS** procedure on dataset `y2015.sas` reports an error, with the `y2015.sas` having been deleted. Had it not been deleted, we could save a lot time, because the PROC MEANS procedure calculating on `y2015.sas` only takes about 30 minutes. However, the DATA step may cost you several hours (due to the limitation on disk I/O speed).
<table>
<tr><th>DATA & PROC Step</th><th>Time Consumed(Estimated)</th></tr>
<tr>
<td>

```sas
data y2015; set
    lib.year2015;
    <WHERE Statement>;
    <IF Statement>;
run;
```

</td><td>10 hours</td>
</tr>
<tr>
<td>

```sas
proc means data=y2015 nway noprint;
    class <all categorical variables>;
    var <all numeric variables>;
    output out=result2015 sum=;
run;
```

</td><td>30 mins</td>
</tr>
</table>

**Solution:** So we have to avoid deleting critical dataset if there exsit errors on the DATA STEP of PROCEDURE above. The action of deleting dataset is so DANGEROUS that we should pay more attention to it.

As mentioned above, **SYSERR** automatic macro variable is reset at each step boundary. It is suggested to create a **global macro variables** at the beginning of the program:
```sas
%let error_exist = 0;
```

Then create a macro functions. The value of error_exist would be turned to TRUE (larger than 0) as soon as the SAS log reports an error.
```sas
%macro check_for_errors;
    %global error_exist;
    %if &syserr > 0 %then %do;
        %let error_exist = 1;
    %end;
%mend check_for_errors;
```

Remeber to call this function after after each DATA STEP or PROC STEP(procedure):
```sas
<DATA STEP>
 %check_for_errors;

<PROCEDURE>
 %check_for_errors;
```

DANGEROUS actions are triggered by the value of global macro variable `error_list`.
```sas
/*DANGEROUS ACTION!!! CHECK error_exist first*/
%macro func;
    %if &error_exist = 0 %then %do;
        proc datasets lib=work; delete y2010; run;
    %end;
%mend func;
%func; 
```

