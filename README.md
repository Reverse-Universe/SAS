## Not Just SAS
This repository is my notebook for tricks and tips that I summarize during work. Different from SAS Programming Guide and SAS official documentation, each note starts with a specific problem that I have met with. I will avoid making a list of the SYNTAX of every STATEMENT and ARGUMENT occurred in the SAS program. In order to make the explanation much more clear, I try to simplify the original problem and refine the key to that. The original intention for creating this notebook is helping reveiw not only the SAS techniques, but also the tricks for data analysis in general.

>Life is Short, I use Python. <img src="https://github.com/Reverse-Universe/SAS/blob/main/MarkdownPicture/python_logo.png" align="middle" width="20%" height="20%"><br>
Dataset is Big, I use SAS. <img src="https://github.com/Reverse-Universe/SAS/blob/main/MarkdownPicture/sas_logo.png" align="middle" width="10%" height="10%">

## The advantages of SAS

### Handling big structured data
&ensp;&ensp;If you read datasets of 10 years, 10 GB for each month, 1200 GB in total. The Python interpreter will immediately reports MemoryError:  `Unable to allocate array with shape (32768, ) and data type int64` for out-of-memory on the computer of *4GB* memory. Fortunately, This is not a problem for SAS, since SAS does not read data from disk to memory, all data is operated on the disk. This means that every **DATA** and **PROC** command in SAS performs both disk I/O reads and writes. **That is why SAS is more robust and scalable than other Data Analysis Tools like Python and R.** Basic R and Python performs operations with everything in memory, whereas SAS doesn't necessarily. SAS is significantly more performant than pure R or Python

### More flexible
&ensp;&ensp;Undoubtedly, MySQL can also manage big structured data by using DQL (Data Query Language). However, the DATA STEP of SAS is more flexible than DQL in MySQL, and the syntax of DATA STEP is much easier to learn and understand. What's more, the PROC SQL, as a powerful procedure in SAS, is quite similar to DQL, so you don't have to learn SAS syntax, since the SAS supports SQL language well.

Here are some examples to show the flexibility of SAS:

**IN= Data Set Option**
>Creates a Boolean variable that indicates whether the data set contributed data to the current observation.

```sas
/* We want to add new variables called 'year' and 'quarter', the values of which are determined by the SAS dataset file that we read from disk. */
data test; set
    /* year2018m01 refers to the dataset of Januray of year 2018 */
    /* y18q1 refers to the first quarter of year 2018 */
    y2018m01(in=y18q1) y2018m02(in=y18q1) y2018m03(in=y18q1)
    y2018m04(in=y18q2) y2018m05(in=y18q2) y2018m06(in=y18q2)
    y2018m07(in=y18q3) y2018m08(in=y18q3) y2018m09(in=y18q3)
    y2018m10(in=y18q4) y2018m11(in=y18q5) y2018m12(in=y18q4)

    y2019m01(in=y19q1) y2019m02(in=y19q1) y2019m03(in=y19q1)
    y2019m04(in=y19q2) y2019m05(in=y19q2) y2019m06(in=y19q2)
    y2019m07(in=y19q3) y2019m08(in=y19q3) y2019m09(in=y19q3)
    y2019m10(in=y19q4) y2019m11(in=y19q5) y2019m12(in=y19q4)

    y2020m01(in=y20q1) y2020m02(in=y20q1) y2020m03(in=y20q1)
    y2020m04(in=y20q2) y2020m05(in=y20q2) y2020m06(in=y20q2)
    y2020m07(in=y20q3) y2020m08(in=y20q3) y2020m09(in=y20q3)
    y2020m10(in=y20q4) y2020m11(in=y20q5) y2020m12(in=y20q4)
    ;

    /* year of data */
    if y18q1 or y18q2 or y18q3 or y18q4 then year=2018;
    if y19q1 or y19q2 or y19q3 or y19q4 then year=2019;
    if y20q1 or y20q2 or y20q3 or y20q4 then year=2020;

    /* quarter of data */
    if y18q1 or y19q1 or y20q1 then quarter=1;
    if y18q2 or y19q2 or y20q2 then quarter=2;
    if y18q3 or y19q3 or y20q3 then quarter=3;
    if y18q4 or y19q4 or y20q4 then quarter=4;
run;
```

**FIRST. & LAST. in the BY Statement**
The SAS's [official documentation](https://documentation.sas.com/?cdcId=vdmmlcdc&cdcVersion=8.11&docsetId=lestmtsref&docsetTarget=p0yeyftk8ftuckn1o5qzy53284gz.htm&locale=en) notes that:
>SAS assgin the following values to FIRST.*variable* and LAST.*variable*:<br>
&ensp;&ensp;**·** FIRST.*variable* has a value of 1 when the current observation is the first observation that is read from the dataset.<br>
&ensp;&ensp;**·** LAST. *variable* has a value of 1 when the current observation is the last observation that is read from the dataset.

>SAS identifies the beginning and end of a BY group by creating two temporary variables for each BY variable: FIRST.variable and LAST.variable. The value of these variables is either 0 or 1. SAS sets the value of FIRST. variable to 1 when it reads the first observation in a BY group, and set the values of LAST.variable to 1 when it reads the last observation in a BY group. These temporary variables are available for DATA step programming but are not added to the output data set.

DO NOT FORGET to sort the dataset first, and the variables in the BY Statement should keep the same as that in the PROC SORT procedure.
```sas
proc sort data=dataset; by a b; run;

data dataset;
    set dataset;
    by a b;
    first_a = first.a;
    last_a = last.a;
    first_b = first.b;
    last_b = last.b;
run;
```

The mechanism of **FIRST.** and **LAST.**

<img src="https://github.com/Reverse-Universe/SAS/blob/main/MarkdownPicture/FIRST%26LAST.png" width="50%" height="50%">

There are five BY groups in the dataset above:
1. a=1 b=1;
2. a=1 b=2;
3. a=1 b=3;
4. a=2 b=1;
5. a=2 b=2.

Variable `a` has been divided into two groups (a=1 and a=2). The value of FIRST.a and LAST.a are used to mark the head and tail of each group.

Varialbe `b` has been divided into five groups (b=1 b=2 b=3 b=1 b=2) depending on the value of `a` (because `b` is behind `a` in the BY Statement). The value of FIRST.b and LAST.b are used to mark the head and tail of each group.


### More reliable
SAS is quite popular among healthcare, finance enterprise and government, because if they make mistakes due to software bugs and it comes to lawsuits, SAS will recompensate them. However, R comes without warranty. Just consider the balance between risk and return. Changing to R may cut down costs on software, but enlarge the risk exposure.

### The solution at your finger tips


### Other merits of SAS
1. [Why pick SAS for your analytics services](https://seleritysas.com/blog/2019/05/10/why-pick-sas-for-your-analytics-services/#:~:text=SAS%20is%20the%20best%20analytics,data%20in%20the%20same%20way.)
2. [Top 8 Reasons to Learn SAS](https://www.jigsawacademy.com/top-8-reasons-to-learn-sas/)
