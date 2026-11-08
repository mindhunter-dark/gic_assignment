Bronze layer:::
Description::
-- We should strive to keep the bronze layer as close to the source as possible. Other than
    metadata columns like record_created_on and record_source, we should refrain from creating any business columns or performing transformations on the data
-- The data is also to be appended to the tables and not overwritten so that we have exact
    copies of the source data from the past. This helps us trace back any issues that we encounter in the silver and gold layers to the corresponding records in the bronze tables

Tables::
course:
-- course_id nvarchar
-- name nvarchar
-- description nvarchar
-- prerequisites nvarchar
-- record_created_on timestamp
-- record_source nvarchar
enrolments:
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