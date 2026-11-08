Bronze layer:::
Design::
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
enrollment:
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
-- One enrollment_id will only have one course_id attached to it. In other words, enrollees
    will have to enroll to two different courses separately (with different enrollment_ids)

Processing from Bronze to Silver::
Notes::
-- We will perform most of the transformations, cleaning steps and data quality checks in 
    the processing step between Bronze and Silver layers. We also have the opportunity to
    normalize the tables for a cleaner schema
-- A data model consisting of dimension and fact tables, with the enrollments fact table 
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
-- While processing the data from the bronze tables, we may encounter a lot of "duplicates"
    as we append the data to these tables rather than overwriting it. Hence, there may 
    exist a row for the same course in the course table for every ingestion we do. We need
    to de-duplicate the data from the bronze layer by using the row_number function
    partitioned on the respective primary keys of the table and selecting the latest 
    record from each partition using the record_created_on field
Data Quality Checks::
-- These happen in between the Bronze and Silver layers. Any records that fail these checks
    may have to be quarantined and alerted to the source for rectification
-- Whenever we quarantine a record in the flow below, we do not propagate it to the next
    step of the pipeline (in this case, from Bronze to Silver), and hold it in a quarantine
    table in the Bronze/Silver layer. For this example, let's create the tables in the
    Silver layer
-- Please find the quarantine tables' definition at the end of the Silver layer defintion.
    The only additional field would be the 'failed_check' field which contains the Data 
    Quality check's description and why that particular row failed this check. Only
    quarantined rows will exist in these tables. A row in a bronze table can have multiple
    rows in its quarantine table if it fails multiple checks

-- In table course:
    -- Check each course_id for a valid value (integer and not null)
        -- If fails, quarantine and alert to source
    -- Check if the prerequisites field is properly built (a string representation of an
        array of course_ids separated by a ', ') and contains valid IDs (each id in the array has to be another course_id in the bronze course table)
        -- If fails, alert to source but continue ingesting this record into Silver as this 
            field may not be too useful when reporting KPIs
-- In table participant:
    -- Check each enrollment_id for a valid value (integer and not null)
        -- If fails, quarantine and alert to source
    -- Check each participant_id for a valid value (integer and not null)
        -- If fails, quarantine and alert to source
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
        -- If not parseable, quarantine and alert to source as this is an important metric.
            If NULL, continue ingestion
    -- Check if credits_used is valid and can be parsed into a decimal value, or is null 
        (as the enrollee may not have had nay credits left)
        -- If not parseable, quarantine and alert to source as this is an important metric.
            If NULL, continue ingestion
Transformations::
-- Data casting and tranformations
    -- In table dim_course
        -- course_id to be casted as an integer (from course)
    -- In table dim_participant
        -- participant_id to be casted as an integer (from enrollment)
    -- In table map_prerequisite
        -- Create the field prerequisite_id (Surrogate key to act as a primary key for the 
            table, auto increment)
        -- Create the field prerequisite_course_id (prerequisites to be split into its 
            individual prerequisite ids and then a row to be created for each such id and cast this id to an integer (from course))
    -- Create table ref_date
        -- Create once at the start of/before pipeline creation and use subsequently
    -- In table fact_enrollment
        -- enrollment_id to be casted as an integer (from enrollment)
        -- participant_id to be casted as an integer (from enrollment)
        -- course_id to be casted as an integer (from enrollment)
        -- Create the field course_date_id (Parse the course_date field from enrollment into 
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
ref_date:
-- date_id integer (Surrogate Key)
-- date date

Fact tables::
fact_enrollment:
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

Quarantine tables::
qua_course:
-- course_id nvarchar
-- name nvarchar
-- description nvarchar
-- prerequisites nvarchar
-- failed_check nvarchar
-- record_created_on timestamp
-- record_created_on_bronze timestamp (From bronze)
-- record_source nvarchar (From bronze)
qua_enrollment:
-- enrollment_id nvarchar
-- participant_id nvarchar
-- participant_name nvarchar
-- course_id nvarchar
-- course_date nvarchar
-- amount nvarchar
-- subsidy nvarchar
-- credits_used nvarchar
-- failed_check nvarchar
-- record_created_on timestamp
-- record_created_on_bronze timestamp (From bronze)
-- record_source nvarchar (From bronze)


Gold Layer:::
Notes::
-- The design of the gold layer largely depends on the business needs. Each specific 
    business need may need a separate gold layer table. Although, we can try to merge
    different needs into a single table is they require similar fields
-- For faster analytics and data-serving, we will need pre-aggregated and de-normalized     
    tables. We can stray away from the rigidity of the Silver layer and think about better column abstacrtion as some of the fields in the Silver layer may not be relevant to the 
    business users
-- For efficient querying and performance optimization, we can index the tables in the 
    Silver and Gold layers on frequently used columns and partition them
    -- In our gold tables below, the course_joined_year + course_joined_month can be a
        good way to partition the tables

Questions to Answer::
-- Let's consider the following questions to solve and then we can design the gold layer 
    tables which can be used to answer these
-- 1. The user wants to see the metrics across the years broken down by calendar 
    month
Tables::
monthly_performace:
-- course_joined_year integer
-- course_joined_month string
-- total_amount float
-- subsidy float
-- credits_used float
-- amount_paid_by_enrollee float
-- no_of_enrollments int
-- no_of_unique_enrollments int
-- no_of_courses int
-- no_of_participants int
Design::
-- Once we have the schema of the table in mind, we can work backwards to build the table
    from the existing Silver layer tables
-- Inner join fact_enrollment to ref_date on course_date_id and date_id
-- Derive the course_joined_year and course_joined_month using the 'date' field
    from ref_date
-- Filter out unknown courses using course_id = -1 if the user wants the numbers to have 
    absolute certainity on where they originate from
-- Aggregate on course_joined_year and course_joined_month and derive the following
    -- total_amount as sum of amount
    -- subsidy as sum of subsidy
    -- credits_used as sum of credits_used
    -- amount_paid_by_enrollee as sum of amount_paid_by_enrollee
    -- no_of_enrollments as count distinct of enrollment_id
    -- no_of_unique_enrollments as count distinct of enrollment_id + participant_id
    -- no_of_courses as count distinct of course_id
    -- no_of_participants as count distinct of participant_id

-- 2. The user wants to see the performance of each course over a time period
Tables::
course_360:
-- course_joined_year integer
-- course_joined_month string
-- course_id integer
-- course_name nvarchar
-- course_description nvarchar
-- total_amount float
-- subsidy float
-- credits_used float
-- amount_paid_by_enrollee float
-- no_of_enrollments int
-- no_of_unique_enrollments int
-- no_of_participants int
Design:
-- Inner Join fact_enrollment to ref_date on course_date_id and date_id
-- Derive the course_joined_year and course_joined_month using the 'date' field
    from ref_date
-- Filter out unknown courses using course_id = -1 if the user wants the numbers to have 
    absolute certainity on where they originate from
-- Inner Join the result to dim_course on course_id
-- Aggregate on 
    course_joined_year,
    course_joined_month,
    course_id,
    course_name and
    course_description
    and derive the following
    -- total_amount as sum of amount
    -- subsidy as sum of subsidy
    -- credits_used as sum of credits_used
    -- amount_paid_by_enrollee as sum of amount_paid_by_enrollee
    -- no_of_enrollments as count distinct of enrollment_id
    -- no_of_unique_enrollments as count distinct of enrollment_id + participant_id
    -- no_of_participants as count distinct of participant_id

-- 3. The user wants to analyze the habits of different participants
participant_360:
-- course_joined_year integer
-- participant_id integer
-- participant_name nvarchar
-- total_amount float
-- subsidy float
-- credits_used float
-- amount_paid_by_enrollee float
-- no_of_enrollments int
-- no_of_unique_enrollments int
-- no_of_courses int
Design:
-- Inner Join fact_enrollment to ref_date on course_date_id and date_id
-- Derive the course_joined_year using the 'date' field from ref_date
-- Inner Join the result to dim_participant on participant_id
-- Aggregate on 
    course_joined_year,
    participant_id and
    participant_name
    and derive the following
    -- total_amount as sum of amount
    -- subsidy as sum of subsidy
    -- credits_used as sum of credits_used
    -- amount_paid_by_enrollee as sum of amount_paid_by_enrollee
    -- no_of_enrollments as count distinct of enrollment_id
    -- no_of_unique_enrollments as count distinct of enrollment_id + participant_id
    -- no_of_courses as count distinct of course_id

Final thoughts::
-- Of course, this is only a small fraction of what can be done in the Data Lake/Medallion 
    gold layer. This layer should be highly cutomizable and rightfully so as the business 
    needs are constantly evolving and so should the layer that feeds reporting and 
    analyses
-- The gold layer may have a lot of tables, each created for a different business facet 
    (Sales, Marketing, Planning etc.,.) and the access to the tables may only be granted to 
    the teams that are allowed to view the data within them