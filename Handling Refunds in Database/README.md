# Handling Refunds in the database
Let's suppose that our database has a table holding millions of healh care bills (or transactions), each of which contains the total amount of money that one patient has paid for the healthcare he received each time he went to the doctor. 

|serial_number|insured_people_ID|payment|
|-------------|-----------------|-------|
|001|A|100|
|002|B|200|
|003|C|300|
|...|...|...|

*Notes: The serial_number is the unique key for the table*

However, the complexitied that are inherent in healthcare payments makes the refund transaction inevitable, in other words, giving the total amount of money back to patient's account and recalculate his deductible (in some cases we won't do the recalculation because inquring the payment of other bills may consume lots of time). Some of the reasons for REFUND include:
+ There was an error in the billing process
+ The price of the medicine was not up-to-date
+ The Social Medical Insurance Fund's Personal Account Balance had changed during the billing process, so the decomposition of the total payment may also change. For more details, please see the *Shanghai Basic Medical Insurance Scheme*.
+ The type of medical treatment (inpatient, outpatient, ...) was wrong. Different kinds of medical treatment correspond to different health insurance schemes, along with different reimbursement rates. If a patient receives different kind of medical treatments, they should be divided into different bills.

The easiest way to add refund transaction to the table is allowing the `payment` to be negative. For example, the transactions with `serial_number` being *'002'* has gone awry, so we have to give 200 yuan back to the account of insured people named *'B'*. Find the wrong bill, then just remain the value of other columns as they are, and let the payment be negative.

|serial_number|insured_people_ID|payment|
|-------------|-----------------|-------|
|001|A|100|
|002|B|200|
|003|C|300|
|**004**|**B**|**-200**|
|...|...|...|

**Shortcoming 1**: All monetary variables should be kept as positive values. Negative values may cause some errors and bugs hard to detect.<br>
<br>
**Solution and Improvement 1**:<br>
Add a `Refund` column that would contain a "1" if it is a refund, or a "0" for a common transcation.

|serial_number|insured_people_ID|payment|Refund|
|-------------|-----------------|-------|------|
|001|A|100|0|
|002|B|200|0|
|003|C|300|0|
|**004**|**B**|**200**|**1**|
|...|...|...|...|

**Shortcoming 2**: The table lacks the marks for original payments which correspond to the refunds.<br>
<br>
**Solution and Improvement 2**:<br>
Add a `Refunded` column that would contain a "1" if this transcation has refunded and becomes invalid. The value of `Refunded` for a negative transcation (refund) is designated to '0'. You could regard the `Refunded` as a mark for validation ('1' for invalid).

|serial_number|insured_people_ID|payment|Refund|Refunded|
|-------------|-----------------|-------|------|--------|
|001|A|100|0|0|
|002|B|200|0|1|
|003|C|300|0|0|
|**004**|**B**|**200**|**1**|0|
|...|...|...|...|...|

If you want to calculate the total expenses of each patient:
```sas
data transation_2;
    set transaction;
    where refund = 0 and refunded = 0;  /* drop the refund and refunded records */
run;

proc means data=transaction_2 nway noprint;
    class insured_people_ID;
    var payment;
    output out=result sum=;
run;
```

**Shortcoming 3**: We don't know the original transaction to which the refund corresponds.<br>
<br>
**Solution and Improvement 3**:<br>
Add `original_serial_number` and `reason_for_refund` columns to the right of table. The value of `original_serial_number` comes from the table itself.

|serial_number|insured_people_ID|payment|Refund|Refunded|original_serial_number|reason_for_refund|
|-------------|-----------------|-------|------|--------|---|----|
|001|A|100|0|0|||
|002|B|200|0|1|||
|003|C|300|0|0|||
|**004**|**B**|**200**|**1**|**0**|**002**|**Reason**|
|...|...|...|...|...|...|...|

**Shortcoming 4**: If we want to analyze refund data exclusively, we have to extract refunds from the whole table firstly.<br>
<br>
**Solution and Improvement 3**:<br>
Strip the refunds from the table (**that is universal practice for database design**).

+ Transaction Table:

|serial_number|insured_people_ID|payment|Refunded|
|-------------|-----------------|-------|--------|
|001|A|100|0|
|002|B|200|1|
|003|C|300|0|

+ Refund Table:

|refund_serial_number|payment|original_serial_number|reason_for_refund|
|--------------------|-------|----------------------|-----------------|
|004|200|002|Reason|



