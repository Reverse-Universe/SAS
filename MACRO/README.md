#  Batch Processing with SAS®: reading consecutive SAS files

## Incremental data by month
Raw data are stored on the server as SAS files. Each file hold the records of a whole month, and the data of different types of patients are stored separately. The reason for storing the raw data by month is that, for example, the data of outpatient and emergency bills may eat up hardware capacity of about 3-5 GBs every month. The table below shows how data becoming larger and larger over time.
|YearMonth|SAS file|number of rows|number of columns|Size|
|---------|--------|--------------|-----------------|----|
|2003/01|OutpatientEmergency0301.sas|10000000+ rows|20+ columns|1.8 GB|
|2009/12|OutpatientEmergency0912.sas|20000000+ rows|20+ columns|3.6 GB|
|2019/12|OutpatientEmergency1912.sas|40000000+ rows|20+ columns|4.2 GB|

*Notes:not include index files*

+ **Not too LARGE** Keep the size of each SAS file within 10GB, and avoid one single file being too big to handle with.

+ **Keep Everything INTACT** The data in the SAS files are not first-handed. At the beginning of each month, we extract fresh data (incremental data) collected last month from data warehouse (in the form of `.txt`), and tranform them into SAS files. So the SAS files would be updated every month (not every day, because it takes some time for the accuracy of data to be verified before extracted from the data warehouse). Every month's fresh data (SAS files) would be under the path of `D:\data\YearMonth\`, for example, the folder `D:\data\202001\` is used to store the data of Janurary 2020, which recording all the bills incurred in that month. Keeping everything intact means not to merge data, instead, convert one txt file to one sas file at a time. If some `.txt` files have some problems and have to be converted into SAS format after being fixed, we could avoid reading irrelevant files. Finally, the amount of I/O that is required could be reduced.
## Directories and SAS files
The name of each sas file contains the year and month of the data, in the format of `4-length digital`: '2001' is for 'Januray,2020'.
|Directory|Year and Month|SAS files|
|---------|--------------|---------|
|D:\data\201001\\ |Jan2010|<ul><li>OutpatientEmergency1001.sas</li><li>Hospitalization1001.sas</li><li>ICU1001.sas</li><li>CriticalIllness1001.sas</li><li>Pharmacy1001.sas</li><li>InternalHos1001.sas</li></ul>|
|D:\data\202012\\ |Dec2020|<ul><li>OutpatientEmergency2012.sas</li><li>Hospitalization2012.sas</li><li>ICU2012.sas</li><li>CriticalIllness2012.sas</li><li>Pharmacy2012.sas</li><li>InternalHos2012.sas</li></ul>|

## Primitive code without MACRO
**Target:** Reading the data from Jan2010 to Dec2020.
```sas
libname lib1001 'D:\data\201001';
libname lib1002 'D:\data\201002';
...
libname lib2012 'D:\data\202012';

data transcation_data;
set
/* Jan2010 */
lib1001.OutpatientEmergency1001
lib1001.Hospitalization1001
lib1001.ICU1001
lib1001.CriticalIllness1001
lib1001.Pharmacy1001
lib1001.InternalHos1001

/* Feb2010 */
lib1002.OutpatientEmergency1002
lib1002.Hospitalization1002
lib1002.ICU1002
lib1002.CriticalIllness1002
lib1002.Pharmacy1002
lib1002.InternalHos1002

...

/* Dec2020 */
lib1002.OutpatientEmergency2012
lib1002.Hospitalization2012
lib1002.ICU2012
lib1002.CriticalIllness2012
lib1002.Pharmacy2012
lib1002.InternalHos2012
;
<WHERE STATEMENT>;
...
run;
```

## Tricks to write more concise code in SAS
Here is an [example](https://documentation.sas.com/?cdcId=pgmsascdc&cdcVersion=9.4_3.5&docsetId=mcrolref&docsetTarget=n01vuhy8h909xgn16p0x6rddpoj9.htm&locale=en) from *SAS® 9.4 and SAS® Viya® 3.5 Programming Documentation*. 

```sas
%macro date_loop(start,end);
    %let start=%sysfunc(inputn(&start,anydtdte9.));
    %let end=%sysfunc(inputn(&end,anydtdte9.));
    %let dif=%sysfunc(intck(month,&start,&end));
    %do i=0 %to &dif;
        %let date=%sysfunc(intnx(month,&start,&i,b),date9.);
        %put &date;
    %end;
%mend date_loop;

%date_loop(01jul2015,01feb2016)
```

    LOG:
    01JUL2015
    01AUG2015
    01SEP2015
    01OCT2015
    01NOV2015
    01DEC2015
    01JAN2016
    01FEB2016

Let's have a look that the functions **INPUTN()** and **INTCK()** used above.

**INPUTN**
> Read a character value using an informat.

**INTCK(interval, start-date, end-date, <'method'>)**
> Returns the number of interval boundaries of a given kind that lie between two dates, times, or datetime values.

***interval***

&ensp;&ensp;*specifies the name of the basic interval type. For example, YEAR specifies yearly intervals.*

***start-date***

&ensp;&ensp;*specifies a SAS expression that represents the starting SAS date, time, or datetime value.*

***end-date***

&ensp;&ensp;*specifies a SAS expression that represents the ending SAS date, time, or datetime value.*

## Taking care of the date format
The start date and end date have been put in the dataset below: 
|min_year_month|max_year_month|
|--------------|--------------|
|201001|202012|

Before we assign these two dates to the macro variables, we have to 'modify' the format of date. The `start-date` and `end-date` in INTCK function must have YEAR, MONTH and DAY. However, `min_year_month` and `max_year_month` only have YEAR and MONTH. Try to combine it with the string '01':
```sas
data min_max_year_month; 
    set min_max_year_month; 
    min_YYMMDD = cat(min_year_month,'01'); 
    max_YYMMDD = cat(max_year_month,'01'); 
    keep min_YYMMDD max_YYMMDD; 
run;
```

Then use **CALL SYMPUT** to assgin them to macro variables.
```sas
/* One Columns to One YYMMDD --> One Rows to One YYMMDD */
proc transpose data=min_max_year_month
    out=macro_name_value(rename=(col1=macro_value))
    name=macro_name;
    var min_YYMMDD max_YYMMDD;
run;
```
min_max_year_month has been tranformed to:
| |macro_name|macro_value|
|-|---------|-----------|
|1|min_YYMMDD|20100101|
|2|max_YYMMDD|20201201|

```sas
data macro_name_value;
    set macro_name_value;
    call symput(macro_name, macro_value);
run;

%put the start YYMMDD is: &min_YYMMDD;
%put the end YYMMDD is : &max_YYMMDD;
```

    LOG:
    the start YYMMDD is: 20100101
    the end YYMMDD is : 20201201
The values of `min_YYMMDD` and `max_YYMMDD` are in the expected format.

## The Power of SAS MACRO function
Create the `%YYMM_loop(start, end)` MACRO function to loop through months in the DATA Step, and call it after defined.
```sas
%macro YYMM_loop(start,end);
    %let start = %sysfunc(inputn(&start, anydtdte 8.));
    %let end = %sysfunc(inputn(&end, anydtdte 8.));
    %let dif = %sysfunc(intck(month, &start, &end));

    data transaction_data; set
    /* batch processing: reading transaction data from every month, every type */
    %do i=0 %to &dif;
        %let long_YYMM = %sysfunc(intnx(month, &start, &i, b), YYMMN.);
        %let short_YYMM = %sysfunc(substr(&long_YYMM), 3, 4));
        lib&short_YYMM&dot.OutpatientEmergency&short_YYMM
        lib&short_YYMM&dot.Hospitalization&short_YYMM
        lib&short_YYMM&dot.CriticalIllness&short_YYMM
        lib&short_YYMM&dot.Pharmacy&short_YYMM
        lib&short_YYMM&dot.InternalHos&short_YYMM
    %end;
    ;  /* DON'T FORGET this semicolon in the SET Statement */
    <Other Statement>;
    run;
%mend YYMM_loop;

%YYMM_loop(&min_YYMMDD, &max_YYMMDD);
```

## More Details
For more information about the **Date, Time and Datetime formats**, See [this page](https://documentation.sas.com/?cdcId=vdmmlcdc&cdcVersion=8.1&docsetId=ds2pg&docsetTarget=p0bz5detpfj01qn1kz2in7xymkdl.htm&locale=en) on *SAS® Visual Data Mining and Machine Learning 8.1*.

Some of the most commonly used formats are listed below:
|Type of Language Element|Language Element|Input|Result|
|------------------------|----------------|-----|------|
|Date formats|DDMMYY.|19069|17/03/12|
| |DDMMYYD.|19069|17-03-12|
|Time formats|TOD.|19069|04:53:28|
|Year formats|YYMMD.|19069|2012-03|
| |YYMMN.|19069|201203|
| |YYMMP.|19069|2012.03|
| |YYMMS.|19069|2012/03|
| |YYMMDD.|19069|12-03-17|
| |YYMMDDS.|19069|12/03/17|
|Year/Quarter formats|YYQ.|19069|2012Q1|






