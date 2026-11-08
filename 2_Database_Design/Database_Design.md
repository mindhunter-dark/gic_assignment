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
-- We will perform most of the transformations, cleaning steps and data quality checks in 
    the processing step between Bronze and Silver layers. We also have the opportunity to
    normalize the tables for a cleaner schema
-- A data model consisting of dimension and fact tables, with the enrolments fact table 
    binding the dimensions to the measures (amount, subsidy and credits_used) seems the cleanest way to go. This schema also keeps the data normalized
-- I've also included a mapping table called 'prerequisite' to move the prerequisites
    array in the course table into its own table. Rather than it being an array, it 
    gets split with each prerequisite for each course having its own row in the new table. 
    This makes any joins upstream easier
-- Created a reference table for all date columns. This column is populated with all the 
    dates in a range (sometime in the past to sometime in the future) and we track the ids of the corresponding date in the fact table. We keep adding future dates (let's say, for the next 5 years) at the start of every year. We should keep this table updated
    so that we don't unwantedly miss out on any dates outside of the range
-- I would keep the two metadata columns
    created in the bronze layer for better traceability even though there is a slight 
    overhead on storage. The detailed details of the tables are as follows

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
-- prerequisite_id (Surrogate Key)
-- course_id integer
-- prerequisite_course_id integer (The course_id of the course that is a prerequisite 
    to course_id)
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
-- date_id integer (Foreign Key from ref_date)
-- amount float
-- subsidy float
-- credits_used float
-- record_created_on timestamp
-- record_created_on_bronze timestamp (From bronze)
-- record_source nvarchar (From bronze)