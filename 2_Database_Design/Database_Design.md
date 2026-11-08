Bronze layer:::
Description::
-- We should strive to keep the bronze layer as close to the source as possible. Other than
    metadata columns like record_created_on and record_source, we should refrain from creating any business columns or performing transformations on the data
-- The data is also to be appended to the tables and not overwritten so that we have exact
    copies of the source data from the past. This helps us trace back any issues that we encounter in the silver and gold layers to the corresponding records in the bronze tables
-- I would create the two tables as follows. There are no transformations in the pipeline
    as of this stage. I have added the record_created_on and record_source metadata columns 
    to both the tables

Tables::
course:
-- course_id nvarchar
-- name nvarchar
-- description nvarchar
-- prerequisites nvarchar
-- record_created_on timestamp
-- record_source nvarchar
enrolment:
-- enrollment_id nvarchar
-- participant_id nvarchar
-- participant_name nvarchar
-- course_id nvarchar
-- course_date nvarchar
-- amount nvarchar
-- subsidy nvarchar
-- credits_used nvarchar
-- record_created_on timestamp
-- record_source nvarchar


Silver layer:::
Assumptions::
-- One enrollment id will only have one course_id attached to it. In other words, enrollees
    will have to enroll to two different courses separately

Description::
Schema::
-- We will perform most of the transformations, cleaning steps and data quality checks in 
    the processing step between Bronze and Silver layers. We also have the opportunity to
    normalize the tables for a cleaner schema
-- A data model consisting of dimension and fact tables, with the enrolments fact table 
    binding the dimensions to the measures (amount, subsidy and credits_used) seems the cleanest way to go. This schema also keeps the data normalized
-- The fact table holds all the key measures (amount, subsidy and credits_used) and binds 
    these to the dimensions using the ids (participant_id, course_id and course_date_id).
    This table can handle the many-to-many relationship between participants and courses
-- I've also included a mapping table called 'prerequisite' to move the prerequisites
    array in the course table into its own table. Rather than it being an array, it 
    gets split with each prerequisite for each course having its own row in the new table. 
    This makes any joins upstream easier
-- Created a reference table for all date columns. This column is populated with all the 
    dates in a range (sometime in the past to sometime in the future) and we track the ids of the corresponding date in the fact table. We keep adding future dates (let's say, for the next 5 years) at the start of every year. We should keep this table updated
    so that we don't unwantedly miss out on any dates outside of the range
-- I would keep the two metadata columns
    created in the bronze layer for better traceability even though there is a slight 
    overhead on storage
Data Quality Checks::
-- These happen in between the Bronze and Silver layers. Any records that fail these checks
    may have to be quarantined and alerted to the source for rectification
-- In table course:
    -- Check each course_id for a valid value (integer and not null)
        -- If fails, quarantine and alert to source
    -- Check if the prerequisites field is properly built (a string representation of an
        array of course_ids separated by a ', ') and contains valid IDs (each id in the array has to be another course_id in the bronze course table)
        -- If fails, alert to source but continue ingesting this record into Silver as this 
            field may not be too useful when reporting KPIs
-- In table participant:
    -- Check each enrolment_id for a valid value (integer and not null)
        -- If fails, quarantine and alert to source
    -- Check each participant_id for a valid value (integer and not null)
        -- If fails, alert to source but continue ingesting this record into Silver. Can use
            a placeholder id of -1 for all such records with this field missing. The 
            participant name can be 'Unknown'. Seed this record into the dim_participant 
            table
    -- Check each course_id for a valid value (integer, not null and present in    
        dim_course)
        -- If fails, alert to source but continue ingesting this record into Silver. Can use
            a placeholder id of -1 for all such records with this field missing. The 
            course name can be 'Unknown'. Seed this record into the dim_course 
            table
    -- Check if course_date is valid and can be parsed into a date
        -- If fails, quarantine and alert to source as this is absolutely necessary to 
            measure the different metrics across time
    -- Check if amount is valid and can be parsed into a decimal value
        -- If fails, quarantine and alert to source as this is an important metric
    -- Check if subsidy is valid and can be parsed into a decimal value, or is null (as the
        enrollee may not have been eligible for a subsidy)
        -- If fails, quarantine and alert to source as this is an important metric
    -- Check if credits_used is valid and can be parsed into a decimal value, or is null 
        (as the enrollee may not have had nay credits left)
        -- If fails, quarantine and alert to source as this is an important metric
Transformations::
-- Data casting and tranformations
    -- In table dim_course
        -- course_id to be casted as an integer (from course)
    -- In table dim_participant
        -- participant_id to be casted as an integer (from enrolment)
    -- In table map_prerequisite
        -- Create the field prerequisite_id (Surrogate key to act as a primary key for the 
            table, auto increment)
        -- Create the field prerequisite_course_id (prerequisites to be split into its 
            individual prerequisite ids and then a row to be created for each such id and cast this id to an integer (from course))
    -- Create table ref_date
        -- Create once at the start of/before pipeline creation and use subsequently
    -- In table fact_enrolment
        -- enrolment_id to be casted as an integer (from enrolment)
        -- participant_id to be casted as an integer (from enrolment)
        -- course_id to be casted as an integer (from enrolment)
        -- Create the field course_date_id (Parse the course_date field from enrolment into 
            a date. Then lookup this field to date from ref_date to get the field)
        -- amount, subsidy and credits_used to be casted as a float (from enrolemnt)
        -- Create the field amount_paid_by_enrollee (using the formula:
            amount_paid_by_enrollee = amount - (subsidy + credits_used))

Tables::
Dimension tables::
dim_course:
-- course_id integer (Primary Key)
-- course_name nvarchar
-- description nvarchar
-- record_created_on timestamp
-- record_created_on_bronze timestamp (From bronze)
-- record_source nvarchar (From bronze)
dim_participant:
-- participant_id integer (Primary Key)
-- participant_name nvarchar
-- record_created_on timestamp
-- record_created_on_bronze timestamp (From bronze)
-- record_source nvarchar (From bronze)

Mapping tables::
map_prerequisite:
-- prerequisite_id (Surrogate Key, AUTO INCREMENT)
-- course_id integer (Foreign Key from dim_course)
-- prerequisite_course_id integer (Foreign Key from dim_course. The course_id of the course 
    that is a prerequisite to course_id)
-- record_created_on timestamp
-- record_created_on_bronze timestamp (From bronze)
-- record_source nvarchar (From bronze)

Reference tables::
ref_date
-- date_id integer (Surrogate Key)
-- date date

Fact tables::
fact_enrolment
-- enrollment_id integer (Primary Key)
-- participant_id integer (Foreign Key from dim_participant)
-- course_id integer (Foreign Key from dim_course)
-- course_date_id integer (Foreign Key from ref_date)
-- amount float
-- subsidy float
-- credits_used float
-- amount_paid_by_enrollee float
-- record_created_on timestamp
-- record_created_on_bronze timestamp (From bronze)
-- record_source nvarchar (From bronze)