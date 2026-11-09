The problem:::
Assuming that you are working with a courses table that contains 200 million records. Each course has a description, and your Data Scientist has built a machine learning model that reads a course description that predicts the skillsets required for that course. This model has been deployed as an AWS SageMaker endpoint, meaning your pipeline calls the model via an API to get predictions.

After running the pipeline, you realise it will take 2 months to process all 200 million records — which is far too long. Your task is to identify the bottleneck and explain how you would optimise this pipeline. Consider both how the SageMaker endpoint is being served, and how the script calling the endpoint is written.

Closer look into the problem and some considerations:::
-- Let's calculate how many rows the current pipeline is processing every second.
    ![alt text](Images/image.png)
    ~ 39 rows / sec
    This can be improved
-- The problem states that the ML model reads a course description and predicts the 
    skillsets required. If the API only accepts one course per call, then this could
    be a limitation. We can consider modifying the endpoint to predict multiple courses in 
    a single API call
        -- We will also have to keep in mind the 30s timeout on AWS API Gateway. Even if the
            model can predict a 1000 courses per API call, if it takes longer than 30s, then
            we will be hitting a lot of timeouts and wasting resources. An exercise can be
            performed to check the rate at which the model can generate predictions per 
            second. From here, we can gauge the maximum no of predictions it can make in 
            a single API call (with a 30 second timeout)
-- We would need to check if the AWS Sagemaker endpoint is being served through the AWS API 
    Gateway via an AWS Lambda or if the Gateway is calling the endpoint directly. The 
    direct method could cut the Lambda invocation times out of the equation, thereby 
    reducing latency
-- When it comes to the data, there could be a case where a course is duplicated across
    different course_ids with the same course_description because users have copied the 
    it from the original course. In this case, finding unique course_descriptions in the 
    dataset can help us figure out if the duplicates problem is one that needs to be taken
    into account
-- If our pipeline is reading from a traditional database like an SQL Server, reading 200 
    million records and processing them could take a really long time. If we can export
    the data into AWS S3 (as parquet files or an Iceberg table), then the processing will 
    be much faster
-- Since the volume is so huge, we would be better off processing it in batches. I would
    partition the data in S3 based on the starting character of the course_description.
    This would give us ~26 different partitions of 7.7 million rows (assuming that the
    course_descriptions are pre-processed to remove unnecessary punctuations, like bullet 
    points at the start of the text). If we were to go this route, we should ditch the 
    API Gateway API and do offline processing instead. Once the entire 200 million records
    are processed, then we can utilize the API for incremental processing (new courses and
    courses with updated descriptions)
-- We will also need to check if there is any kind of parallelism involved with the current
    pipeline. If not, then we should consider implementing it. As the predictions for 
    individual course_descriptions are independent from one another, parallelism can vastly 
    improve the pipeline performance
-- 
 

