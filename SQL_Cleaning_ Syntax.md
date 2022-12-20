Original Data Set

![Data_Cleaning_Original](https://user-images.githubusercontent.com/112409778/208680653-eaed806b-ee56-4eae-a27c-9a135a67b48f.png)


- *There are duplicates in the 'Absence_Type' column, there are two absence types for the same day*
- *All column in the dataset isn't relevenat to calculating attendance*

## Step 1: Create a query to return all relevant columns for attendance calculation.
~~~SQL
/* Data Cleanining and table creation with month, excused/unexcused absences*/
SELECT employee_id, date, MONTH(date) AS Month,department_name, absence_type, 
	/*Count of Excused, Unexcused, and Absences based on IF THEN Formula*/
	COUNT(CASE WHEN Absence_Type IN ('Holiday','Personal','Vacation','Personal_(Longevity)', 'Professional_Development', 'Administrative_Leave','Bereavement', 'Religious_Observation', 'Sick', 'Covid_Sick') THEN 'Excused' END) AS Excused,
	COUNT (CASE WHEN Absence_Type = 'Unpaid_Time' THEN 'Unexcused' END) AS Unexcused,
	COUNT (CASE WHEN Absence_Type = 'Absent' THEN 'Absence' END) AS Absence,

	/* CASE Statement for the number of school days per month aliased as SchoolDays*/
	CASE 
	WHEN MONTH(date) = 8 THEN 8 
	WHEN MONTH(date) = 9 THEN 20
	WHEN MONTH(date) = 10 THEN 20
	WHEN MONTH(date) = 11 THEN 17 END SchoolDays

FROM dbo.Annonymized_Data_Clocking



WHERE absence_type <> 'Absent' AND Employee_ID = 284 --  Filter to exclude all Absence_Type 'Absent' this is the duplicate row that makes it difficult to calculate attendance 

GROUP BY Employee_id, date,department_name, absence_type, hours
~~~

![Data_Cleaning_AbsenceCat](https://user-images.githubusercontent.com/112409778/208681303-7d95e2de-9cf4-4bab-ad93-4266f8f3af94.png)

- *Duplicate absences have been removed, only legitimate absences remain*
- *Absence categories, 'Excused' and 'Unexcused', have been added*
- *All the columns present are relevant to the calculation of attendance*
- *Insertion of 'School Days' column which will become the denominator in the calculation of school attendance percentage*



## Step 2: The previous query will become a subquery in the 'FROM' statement, and calculations will perfromed to find employee's absences 

~~~SQL 

SELECT s.employee_id, s.date,s.month,s.SchoolDays,s.department_name, s.absence, s.excused, s.unexcused, s.absence_type, 
(s.unexcused + s.absence) - s.excused AS true_absence
FROM (

SELECT employee_id, date, MONTH(date) AS Month,department_name, absence_type, 
	/*Count of Excused, Unexcused, and Absences based on IF THEN Formula*/
	COUNT(CASE WHEN Absence_Type IN ('Holiday','Personal','Vacation','Personal_(Longevity)', 'Professional_Development', 'Administrative_Leave','Bereavement', 'Religious_Observation', 'Sick', 'Covid_Sick') THEN 'Excused' END) AS Excused,
	COUNT (CASE WHEN Absence_Type = 'Unpaid_Time' THEN 'Unexcused' END) AS Unexcused,
	COUNT (CASE WHEN Absence_Type = 'Absent' THEN 'Absence' END) AS Absence,

	/* CASE Statement for the number of school days per month aliased as SchoolDays*/
	CASE 
	WHEN MONTH(date) = 8 THEN 8 
	WHEN MONTH(date) = 9 THEN 20
	WHEN MONTH(date) = 10 THEN 20
	WHEN MONTH(date) = 11 THEN 17 END SchoolDays

FROM dbo.Annonymized_Data_Clocking



WHERE absence_type <> 'Absent' AND Employee_ID = 284  --  Filter to exclude all Absence_Type 'Absent' this is the duplicate row that makes it difficult to calculate attendance 

GROUP BY Employee_id, date,department_name, absence_type, hours) AS s -- Alias subquery
GROUP BY s.employee_id, s.date,s.month,s.department_name, s.absence, s.excused, s.unexcused, s.absence_type,s.Day_type,s.SchoolDays
~~~

![Data_Cleaning_TrueAbsence](https://user-images.githubusercontent.com/112409778/208683686-1c207193-cdad-47de-98a1-7c3705916efc.png)

- *Creation of a 'true_absence' column, the difference between the sum of 'excused' & 'absence' from 'unexcused'*
- *Negative results in the 'true_absence' column is indicative 'excused' absences and will not negatively impact their attendance percentage*

## Step 3: Create a CTE 

~~~SQL
/* CTE of Cleaned Data*/
WITH Clean AS (

SELECT s.employee_id, s.date,s.month,s.SchoolDays,s.department_name, s.absence, s.excused, s.unexcused, s.absence_type, (s.unexcused + s.absence) - s.excused AS true_absence
FROM (

SELECT employee_id, date, MONTH(date) AS Month,department_name, absence_type, 
	/*Count of Excused, Unexcused, and Absences based on IF THEN Formula*/
	COUNT(CASE WHEN Absence_Type IN ('Holiday','Personal','Vacation','Personal_(Longevity)', 'Professional_Development', 'Administrative_Leave','Bereavement', 'Religious_Observation', 'Sick', 'Covid_Sick') THEN 'Excused' END) AS Excused,
	COUNT (CASE WHEN Absence_Type = 'Unpaid_Time' THEN 'Unexcused' END) AS Unexcused,
	COUNT (CASE WHEN Absence_Type = 'Absent' THEN 'Absence' END) AS Absence,

	/* CASE Statement for the number of school days per month aliased as SchoolDays*/
	CASE 
	WHEN MONTH(date) = 8 THEN 8 
	WHEN MONTH(date) = 9 THEN 20
	WHEN MONTH(date) = 10 THEN 20
	WHEN MONTH(date) = 11 THEN 17 END SchoolDays

FROM dbo.Annonymized_Data_Clocking



WHERE absence_type <> 'Absent' AND Employee_ID = 284  --  Filter to exclude all Absence_Type 'Absent' this is the duplicate row that makes it difficult to calculate attendance 

GROUP BY Employee_id, date,department_name, absence_type, hours) AS s -- Alias subquery
GROUP BY s.employee_id, s.date,s.month,s.department_name, s.absence, s.excused, s.unexcused, s.absence_type,s.Day_type,s.SchoolDays)
~~~

## Step 4: Calculate the attendance percentages of employees

~~~SQL
/* CTE of Cleaned Data*/
WITH Clean AS (

SELECT s.employee_id, s.date,s.month,s.SchoolDays,s.department_name, s.absence, s.excused, s.unexcused, s.absence_type, (s.unexcused + s.absence) - s.excused AS true_absence
FROM (

SELECT employee_id, date, MONTH(date) AS Month,department_name, absence_type, 
	/*Count of Excused, Unexcused, and Absences based on IF THEN Formula*/
	COUNT(CASE WHEN Absence_Type IN ('Holiday','Personal','Vacation','Personal_(Longevity)', 'Professional_Development', 'Administrative_Leave','Bereavement', 'Religious_Observation', 'Sick', 'Covid_Sick') THEN 'Excused' END) AS Excused,
	COUNT (CASE WHEN Absence_Type = 'Unpaid_Time' THEN 'Unexcused' END) AS Unexcused,
	COUNT (CASE WHEN Absence_Type = 'Absent' THEN 'Absence' END) AS Absence,

	/* CASE Statement for the number of school days per month aliased as SchoolDays*/
	CASE 
	WHEN MONTH(date) = 8 THEN 8 
	WHEN MONTH(date) = 9 THEN 20
	WHEN MONTH(date) = 10 THEN 20
	WHEN MONTH(date) = 11 THEN 17 END SchoolDays

FROM dbo.Annonymized_Data_Clocking



WHERE absence_type <> 'Absent' AND Employee_ID = 284 --  Filter to exclude all Absence_Type 'Absent' this is the duplicate row that makes it difficult to calculate attendance 

GROUP BY Employee_id, date,department_name, absence_type, hours) AS s -- Alias subquery
GROUP BY s.employee_id, s.date,s.month,s.department_name, s.absence, s.excused, s.unexcused, s.absence_type,s.Day_type,s.SchoolDays)

/* PossibleDays = Difference of SchoolDays and sum of Excused days, DaysPresent =  PossibleDays minus the sum of unexcused days*/  

SELECT employee_id, department_name,month, SchoolDays, SUM(excused) AS ExcusedTotal, SUM(unexcused) AS UnexcusedTotal,(SchoolDays - SUM(excused)) - SUM(Unexcused) AS DaysPresent, SchoolDays - SUM(excused) AS PossibleDays, 
(((SchoolDays - SUM(excused)) - SUM(Unexcused)) * 1.0/  (SchoolDays - SUM(excused))  * 1.0) * 100 AS AttPercentage --DaysPresent/PossibleDays
FROM Clean -- CTE
GROUP BY employee_id, month, SchoolDays, excused, Unexcused,Department_Name, Day_type
HAVING  (((SchoolDays - SUM(excused)) - SUM(Unexcused)) * 1.00/  (SchoolDays - SUM(excused))  * 1.00) * 100 > 0.00 AND SchoolDays - SUM(excused) > 0
~~~~

![Data_cleaning_Fin](https://user-images.githubusercontent.com/112409778/208684735-dc31aa30-7743-4ad6-946f-d51ff65ad392.png)

- *Creation of a 'DaysPresent' column, the difference between 'SchoolDays' and 'UnexcusedTotal' days for the specific month*
- *Creation of 'PossibleDays' column, the difference between 'SchoolDays' and 'ExcusedTotal' days for the specific month*
- *Calculation of 'AttPercentage', the quotient of the 'DaysPresent' divided by 'PossibleDays'*
