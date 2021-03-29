# Listing Patients Having Appointments

The following example query demonestrates how we can list all details for patients 
having appointments during a specific time range.

```sql
SELECT DISTINCT
  -- Listing patient details
  patient.id, -- the database record serial number 
  patient.chartnumber as "Chart Number",
  patient.firstname as "First Name",
  patient.middlename as "Middle Name",
  patient.lastname as "Last Name",
  patient.email as "Email",
  patient.mobilephonenumber as "Phone Number",
  -- we can use the to_char function to print timestamps in a specific format
  to_char(patient.dateofbirth, 'yyyy-mm-dd') as "Date of Birth"
FROM "crm"."appointment" AS ap
  -- Retriving the appointment status name from reference_value 
  LEFT JOIN "crm"."reference_value" AS statuslist on ap.statusid = statuslist.id
  -- Retriving the appointment type from reference_value
  LEFT JOIN "crm"."reference_value" AS cat on ap.categoryid = cat.id
  -- Retriving details for the machine used for the treatment (can be null)
  LEFT JOIN "crm"."machine" AS m on ap.machineid = m.id
  -- Retriving the branch name
  LEFT JOIN "crm"."branch" AS c on ap.clinicid = c.id
  -- Retriving details for the nurse that the appointment was assigned to
  LEFT JOIN "crm"."employee" AS nurse on ap.nurseid = nurse.id
  -- Retriving details for the supervisor doctor assigned
  LEFT JOIN "crm"."employee" AS doctor on ap.dentistid = doctor.id
  -- Retriving details for each patient for each appointment
  LEFT JOIN "crm"."patient" AS patient on ap.patientkey = patient.key
-- Filtering results based on the selected time range
where $__unixEpochFilter(ap.starttime/1000) 
-- Filtering results based on selected appointment status
and ap.statusid in ($Status)
-- Filtering results based on appointment type
and ap.categoryid in ($Type)
-- Filtering results based on selected machine
and COALESCE(ap.machineid, -1) in ($Machine)
```

Notice how there's no ```GROUP BY``` statment. Yet, no duplicate patient profiles will show becasue 
PostgreSQL groups identical records together by default.
