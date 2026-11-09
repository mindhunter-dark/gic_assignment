The problem:::
Assuming that you are working with a courses table that contains 200 million records. Each course has a description, and your Data Scientist has built a machine learning model that reads a course description that predicts the skillsets required for that course. This model has been deployed as an AWS SageMaker endpoint, meaning your pipeline calls the model via an API to get predictions.

After running the pipeline, you realise it will take 2 months to process all 200 million records — which is far too long. Your task is to identify the bottleneck and explain how you would optimise this pipeline. Consider both how the SageMaker endpoint is being served, and how the script calling the endpoint is written.

Further dive into the problem:::
-- Let's calculate how many rows the current pipeline is processing every second.
    ![alt text](Images/image.png)
    ~ 39 rows / sec

