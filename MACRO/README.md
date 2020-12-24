#  Batch Processing with SAS®: reading consecutive SAS files


Raw data are stored on the server as SAS files. Each file hold the records of a whole month, and the data of different types of patients are stored separately. The reason for storing the raw data by month is that, for example, the data of outpatient and emergency bills may eat up hardware capacity of about 3-5 GBs every month. The table below shows how data becoming larger and larger over time.
|YearMonth|SAS file|number of rows|number of columns|Size|
|---------|--------|--------------|-----------------|----|
|2003/01|OutpatientEmergency0301.sas|10000000+ rows|20+ columns|1.8 GB|
|2009/12|OutpatientEmergency0912.sas|20000000+ rows|20+ columns|3.6 GB|
|2019/12|OutpatientEmergency1912.sas|40000000+ rows|20+ columns|4.2 GB|
*Notes:not include index files*

+ **Not too LARGE** Keep the size of each SAS file within 10GB, and avoid one single file being too big to handle with.
+ **Keep Everything INTACT** The data in the SAS files are not first-hand. At the beginning of each month, we extract fresh data collected last month from data warehouse (in the form of `.txt`), and tranform them into SAS files. So the SAS files would be updated every month (not every day, because it takes some time for the accuracy of data to be verified before extracted from the data warehouse). Every month's fresh data (SAS files) would be under the path of `d:\YearMonth\`, for example, the folder `d:\202001\` is used to store the data of Janurary 2020, which recording all the bills incurred in that month. Keeping everything intact means not to merge data, instead, convert one txt file to one sas file at a time.


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

/* Please input latest year and month */
%let today_YM_long = 202011;

/* Please input raw data */
data raw_data;
input sfzh $18. check_year_s check_year_e;
datalines;
310XXXXXXXXXXXXXXX 20180000 20180999
;
run;

%let today_YM_short = %sysfunc(substr(&today_YM_long,3,4));
%let dot=.;

/* 提取起始年的最小数，结束年的最大数 */
proc sql;
    create table min_max_year_month as
    select put(min(check_year_s), 8.) as min_year_month,
           put(max(check_year_e), 8.) as max_year_month
    from raw_data;
quit;

/* 提取四位数的年月字符串，然后添加1号（为了之后输出到date_loop宏函数中） */
dat min_max_year_month;
    set min_max_year_month;
    min_year_month = cat(substr(min_year_month,1,6), '01');
    max_year_month = cat(substr(max_year_month,1,6), '01');
run;

/*行转列，便于之后的call symput*/
proc transpose data=min_max_year_month
    out=macro_name_value(rename=(col1=macro_value))
    name=macro_name;
    var min_year_month max_year_month;
run;

/* 最小年月和最大年月分别赋值给宏变量minyear和maxyear */
data macro_name_value;
    set macro_name_value;
    call symput(macro_name, macro_value);
run;

%put min year month = &min_year_month;
%put max year month = &max_year_month;


%macro date_loop_lib(start,end);
%let start=%sysfunc(inputn(&start, anydtdte 8.));
%let end=%sysfunc(inputn(&end, anydtdte 8.));
%let dif=%sysfunc(intck(month, &start, &end));

data jsbx; set
/*批量libname*/
%do i=0 %to &dif;
%let long_YearMonth = %sysfunc(intnx(month, &start, &i, b), YYMM.);
%let short_YearMonth = %sysfunc(substr(&long_YearMonth), 3, 4));
y&short_YearMonth&dot.mz&short_YearMonth
y&short_YearMonth&dot.zy&short_YearMonth
...
%end;
;

where ...
...
run;

%mend date_loop_lib;
%date_loop_lib(&min_year_month, &max_year_month)




