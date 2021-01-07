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

