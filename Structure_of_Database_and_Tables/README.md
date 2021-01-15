# The Structure of Charge Datasets

**ATTENTION:**
Please click [**here**](../Shanghai_Basic_Medical_Insurance_Scheme/EN.md) to learn about **Shanghai Social Medical Insurance Policy** before you read this article.

## Table 1: Total expenses Data
one observation refers to one medical bill<br><br>
**Monetary Variables:**
|Column Name|Description|Type|
|-----------|-----------|----|
|Covered_Expenses|Covered Charges|Numeric|
|Uncovered_Expenses|Uncovered Charges|Numeric|
|Class_C_Expenses|Expenses of Class C|Numeric|
|Extra_Expenses|Extra Payment|Numeric|
|Co_pay_Expenses|Co-pay|Numeric|
||||
|Registration_Fee||Numeric|
|Diagnostics_Fee||Numeric|
|Treatment_Fee||Numeric|
|Surgical_Materials_Fee||Numeric|
|Bed_Fee||Numeric|
|Nursing_Fee||Numeric|
|Examination_Fee||Numeric|
|Laboratory_Fee||Numeric|
|Medical_Imaging_Fee||Numeric|
|X-ray_Fee||Numeric|
|Blood_Transfusion_Fee||Numeric|
|Oxygen_Therapy_fee||Numeric|
|Medication_Fee||Numeric|
|Chinese_patent_medicine_Fee||Numeric|
|Herbs_Fee||Numeric|
|Other_Fee||Numeric|

## Table 2: Covered Charges Data

**Monetary Variables:**
|Column Name|Description|Type|
|-----------|-----------|----|
|Recent_Acc_Payment|Paid by Surplus Capital from Recent Years' Individual Account|Numeric|
||||
|Deductible_Prev_Acc_Payment|Expenses Below Deductible Line : Paid by  Surplus Capital from Previous Years' Individual Account|Numeric|
|Deductible_OOP|Expenses Below Deductible Line : Paid by cash (OOP)|Numeric|
||||
|Co_Reim|Expenses Above Deductible Line and Below Cap Line : Paid by Funds (Pooling or Local Additional)|Numeric|
|Co_Prev_Acc_Payment|Expenses Above Deductible Line and Below Cap Line : Paid by Surplus Capital from Previous Years' Individual Account|Numeric|
|Co_OOP|Expenses Above Deductible Line and Below Cap Line : Paid by Cash (Coinsurance Payment)|Numeric|
||||
|Cap_Addi_Fund|Expenses Above Cap Line : Paid by Local Additional Fund|Numeric|
|Cap_OOP|Expensed Above Cap Line : Paid by Cash (OOP)|Numeric|

**_Notes_:**
+ *For expenses of outpatient and emergency medical treatments, the amount above deductible line and below cap line is paid by Local Additioanl Fund. But when it comes to the expenses of medical treatment of hospitalization or stay in emergency office for observation, that part of amount is paid by Pooling Fund.*

+ *The expenses of outpatient and emergency medical treatments do not have cap, unlike that of hopitalization, severe diseases and home sickbed.*

+ *URBMI doesn't have individual accout*
