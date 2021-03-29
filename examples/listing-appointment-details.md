# Listing Appointment Details
The following is a detailed example of how to list details for appointments data.

```sql
SELECT
  ap.id, -- The appointment's serial number in the database table
  ap.patientkey, -- The string identifier of the pantient having the appointment
  p.chartnumber AS "Patient Chart #", -- The patient Profile number
  concat(p.firstname, ' ', p.middlename, ' ', p.lastname) AS "Patient Name", -- The patient's full name
  -- p.fullname AS "Patient Name", -- Another way to print the patient's full name
  cat.name AS "Appointment Type", -- The category is the "appointment type"
  --c.name AS "Clinic", -- The clinic is the company profile (branch)
  --to_char(to_timestamp(ap.date/1000) AT TIME ZONE 'Asia/Dubai', 'yyyy-mm-dd hh:mi AM') AS "Appointment Date",
  to_char(to_timestamp(ap.starttime/1000) AT TIME ZONE 'Asia/Dubai', 'yyyy-mm-dd hh:mi AM') AS "Start Time",
  ap.duration / 60000 AS "Duration (minutes)", -- The duration of the appointment
  to_char(to_timestamp(ap.endtime/1000) AT TIME ZONE 'Asia/Dubai', 'yyyy-mm-dd hh:mi AM') AS "End Time",
  m.name AS "Machine", -- the machine being used for the treatment
  concat(doctor.firstname, ' ' , doctor.lastname) AS "Doctor", -- Supervisor's name
  concat(nurse.firstname, ' ' , nurse.lastname) AS "Nurse", -- Nurse's name
  statuslist.name AS "Status" -- Appointment status ('Confirmed, Canceled, Complete, etc..)
FROM "crm"."appointment" AS ap
  LEFT JOIN "crm"."reference_value" AS statuslist ON ap.statusid = statuslist.id
  LEFT JOIN "crm"."reference_value" AS cat ON ap.categoryid = cat.id
  LEFT JOIN "crm"."machine" AS m ON ap.machineid = m.id
  LEFT JOIN "crm"."branch" AS c ON ap.clinicid = c.id
  LEFT JOIN "crm"."employee" AS nurse ON ap.nurseid = nurse.id
  LEFT JOIN "crm"."employee" AS doctor ON ap.dentistid = doctor.id
  LEFT JOIN "crm"."patient" AS p ON ap.patientkey = p.key
WHERE $__unixEpochFilter(ap.starttime/1000) -- The $__unixEpochFilter lists results only within the time range selected in the dashboard
AND ap.statusid IN ($Status) -- This is an example of using a dashboard variable "$Status" to filter results based on a specific status select in the dashboard by the user
-- If we were to perform filtration using the machine field, it's important to notice
-- that not all appointments has a machine, sometimes this field will be null.
-- That's why we will be replacing the null with -1 and adding a "None" option
-- for the variable with the value -1 to display results with no machine .
AND COALESCE(ap.machineid, -1) IN ($Machine) 
ORDER BY "Start Time"
```

In the above example, appointments are sorted based on the schedule of each appointment. 
If we were to sort them by the time they were created instead, we will have to change 

```sql
WHERE $__unixEpochFilter(ap.starttime/1000)
```

into 

```sql
WHERE $__timeFilter(ap.createddate)
```

and

```sql
ORDER BY "Start Time"
```

to 

```sql
ORDER BY "Created Date"
```

PS: Notice how in this scenario we are using $\__timeFilter rather than $\__unixEpochFilter
and that's based on the datatype of the field we are using for filtration, because $\__timeFilter
takes a timestamp, and the **createddate** is of type **timestamp** while **starttime** is a numeric 
value of type **integer** representing a Unix time in milliseconds. First we divide it by 1000 to convert
it to seconds and then we can use the $\__unixEpochFilter.
