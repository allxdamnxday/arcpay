Federal 2025 Income Tax Withholding: Automated Percentage Method Guide
This guide provides the key data and steps to implement the 

Percentage Method for Automated Payroll Systems, as detailed in Section 1 of IRS Publication 15-T for 2025. This method is the most robust for automated systems as it uses a single, unified worksheet (

Worksheet 1A) to handle both newer Forms W-4 (2020 or later) and legacy Forms W-4 (2019 or earlier).

The core of this method is to annualize wages, calculate an annual tentative withholding amount, and then convert that amount back to the payroll period.

Core Logic: Worksheet 1A Step-by-Step Calculation
This process follows the steps outlined in "Worksheet 1A. Employer's Withholding Worksheet for Percentage Method Tables for Automated Payroll Systems".

Step 1: Adjust the Employee's Wage Amount
First, calculate the employee's annualized wages.


Line 1a: Enter the employee's total taxable wages for the current payroll period.


Line 1b: Determine the number of pay periods in the year from the table below.


| Pay Period | Number of Periods |
| :--- | :--- |
| Daily | 260 |
| Weekly | 52 |
| Biweekly | 26 |
| Semimonthly | 24 |
| Monthly | 12 |
| Quarterly | 4 |
| Semiannually | 2 |


Line 1c: Multiply the wages (Line 1a) by the number of pay periods (Line 1b) to get the Annual Wage Amount.

Next, follow the appropriate path based on the employee's Form W-4 version.

If the employee has a Form W-4 from 2020 or later:

Line 1d: Enter the amount for other income from Step 4(a) of Form W-4.


Line 1e: Add Line 1c and Line 1d.


Line 1f: Enter the deduction amount from Step 4(b) of Form W-4.

Line 1g: If the box in Step 2 of Form W-4 is checked, enter $0. Otherwise, enter 

$12,900 for Married Filing Jointly or $8,600 for all other filing statuses.


Line 1h: Add Line 1f and Line 1g.

Line 1i: Subtract Line 1h from Line 1e. If the result is zero or less, enter $0. This is the 

Adjusted Annual Wage Amount.

If the employee has a Form W-4 from 2019 or earlier:

Line 1j: Enter the number of allowances claimed on the employee's Form W-4.


Line 1k: Multiply the number of allowances (Line 1j) by $4,300.

Line 1l: Subtract Line 1k from the annual wages (Line 1c). If the result is zero or less, enter $0. This is the 

Adjusted Annual Wage Amount.

Step 2: Figure the Tentative Withholding Amount
This step uses the Adjusted Annual Wage Amount from Step 1 and the annual tax tables below.


Line 2a: Enter the Adjusted Annual Wage Amount from either Line 1i or 1l.

Determine Which Table to Use:

If the employee has a Form W-4 from 2019 or earlier, use the 

STANDARD Withholding Rate Schedules.

If the employee has a Form W-4 from 2020 or later, check if the box in Step 2 is checked.

If the box is 

NOT checked, use the STANDARD Withholding Rate Schedules.

If the box 

IS checked, use the Form W-4, Step 2, Checkbox, Withholding Rate Schedules.


Line 2b - 2d: Find the correct row in the appropriate annual table where the Adjusted Annual Wage Amount (Line 2a) is between the values in Column A and Column B. Enter the values from Column A, Column C, and Column D into worksheet lines 2b, 2c, and 2d, respectively.


Line 2e: Subtract Line 2b from Line 2a.


Line 2f: Multiply the result from Line 2e by the percentage on Line 2d.


Line 2g: Add Line 2c and Line 2f.

Line 2h: Divide the result from Line 2g by the number of pay periods (from Line 1b). This is the 

Tentative Withholding Amount.

2025 Annual Percentage Method Tables
STANDARD Withholding Rate Schedules


(Use if Form W-4 is from 2019 or earlier, OR if the box in Step 2 of a 2020 or later Form W-4 is NOT checked) 

Filing Status	Adjusted Wage Over (A)	But less than (B)	Base Amount (C)	Plus % (D)	Of Excess Over (E)
Married Filing Jointly	$0	$17,100	$0.00	0%	$0
$17,100	$40,950	$0.00	10%	$17,100
$40,950	$114,050	$2,385.00	12%	$40,950
$114,050	$223,800	$11,157.00	22%	$114,050
$223,800	$411,700	$35,302.00	24%	$223,800
$411,700	$518,150	$80,398.00	32%	$411,700
$518,150	$768,700	$114,462.00	35%	$518,150
$768,700	and over	$202,154.50	37%	$768,700
Single or Married Filing Separately	$0	$6,400	$0.00	0%	$0
$6,400	$18,325	$0.00	10%	$6,400
$18,325	$54,875	$1,192.50	12%	$18,325
$54,875	$109,750	$5,578.50	22%	$54,875
$109,750	$203,700	$17,651.00	24%	$109,750
$203,700	$256,925	$40,199.00	32%	$203,700
$256,925	$632,750	$57,231.00	35%	$256,925
$632,750	and over	$188,769.75	37%	$632,750
Head of Household	$0	$13,900	$0.00	0%	$0
$13,900	$30,900	$0.00	10%	$13,900
$30,900	$78,750	$1,700.00	12%	$30,900
$78,750	$117,250	$7,442.00	22%	$78,750
$117,250	$211,200	$15,912.00	24%	$117,250
$211,200	$264,400	$38,460.00	32%	$211,200
$264,400	$640,250	$55,484.00	35%	$264,400
$640,250	and over	$187,031.50	37%	$640,250

Export to Sheets
Form W-4, Step 2, Checkbox, Withholding Rate Schedules


(Use if the box in Step 2 of a 2020 or later Form W-4 IS checked) 

Filing Status	Adjusted Wage Over (A)	But less than (B)	Base Amount (C)	Plus % (D)	Of Excess Over (E)
Married Filing Jointly	$0	$15,000	$0.00	0%	$0
$15,000	$26,925	$0.00	10%	$15,000
$26,925	$63,475	$1,192.50	12%	$26,925
$63,475	$118,350	$5,578.50	22%	$63,475
$118,350	$212,300	$17,651.00	24%	$118,350
$212,300	$265,525	$40,199.00	32%	$212,300
$265,525	$390,800	$57,231.00	35%	$265,525
$390,800	and over	$101,077.25	37%	$390,800
Single or Married Filing Separately	$0	$7,500	$0.00	0%	$0
$7,500	$13,463	$0.00	10%	$7,500
$13,463	$31,738	$596.25	12%	$13,463
$31,738	$59,175	$2,789.25	22%	$31,738
$59,175	$106,150	$8,825.50	24%	$59,175
$106,150	$132,763	$20,099.50	32%	$106,150
$132,763	$320,675	$28,615.50	35%	$132,763
$320,675	and over	$94,384.88	37%	$320,675
Head of Household	$0	$11,250	$0.00	0%	$0
$11,250	$19,750	$0.00	10%	$11,250
$19,750	$43,675	$850.00	12%	$19,750
$43,675	$62,925	$3,721.00	22%	$43,675
$62,925	$109,900	$7,956.00	24%	$62,925
$109,900	$136,500	$19,230.00	32%	$109,900
$136,500	$324,425	$27,742.00	35%	$136,500
$324,425	and over	$93,515.75	37%	$324,425

Export to Sheets
Step 3: Account for Tax Credits

Line 3a: If the employee's Form W-4 is from 2020 or later, enter the amount from Step 3. Otherwise, enter $0.


Line 3b: Divide the credit amount on Line 3a by the number of pay periods (from Line 1b).

Line 3c: Subtract Line 3b from the Tentative Withholding Amount (Line 2h). If the result is zero or less, enter $0.

Step 4: Figure the Final Amount to Withhold

Line 4a: Enter the additional amount to withhold from the employee's Form W-4 (Step 4(c) for 2020 or later forms, or Line 6 for earlier forms).

Line 4b: Add Line 3c and Line 4a. This is the 

final amount to withhold from the employee's wages for the pay period.

Special Adjustment for Nonresident Alien (NRA) Employees
Before starting the worksheet above, you must adjust the wages for NRA employees. This procedure applies only to NRA employees performing services within the United States.


Determine the W-4 Version:

If the NRA employee has a Form W-4 from 

2019 or earlier, add the amount from Table 1 below to their wages for the payroll period.

If the NRA employee has a Form W-4 from 

2020 or later, add the amount from Table 2 below to their wages for the payroll period.


Use the Adjusted Wages: Use this new, higher wage amount as the "total taxable wages" on Line 1a of Worksheet 1A.

Note: These additional amounts are used only for calculating income tax withholding. They should not be included in any box on the employee's Form W-2 and do not affect Social Security, Medicare, or FUTA tax liability.


Table 1: Add-Back for NRA Employees with 2019 or Earlier Form W-4 

Payroll Period	Add to Wages
Weekly	$205.80
Biweekly	$411.50
Semimonthly	$445.80
Monthly	$891.70
Quarterly	$2,675.00
Semiannually	$5,350.00
Annually	$10,700.00

Export to Sheets

Table 2: Add-Back for NRA Employees with 2020 or Later Form W-4 

Payroll Period	Add to Wages
Weekly	$288.50
Biweekly	$576.90
Semimonthly	$625.00
Monthly	$1,250.00
Quarterly	$3,750.00
Semiannually	$7,500.00
Annually	$15,000.00
