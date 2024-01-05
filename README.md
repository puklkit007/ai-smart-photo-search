# AI Smart Photo Search
This is a serverless photo album web application, that can be searched using natural language through both text and voice. The architecture diagram is as follow: B1 is a S3 bucket stores front-end stuff, together with API Gateway, LF2 (Lambda Function 2) forming a website that can handle user request (upload photo, search photo by text, search photo by voice, etc.)

A new photo is uploaded to another S3 bucket B2, which that trigger LF1 to parse the photo using AWS Rekognition that returns some keywords of the photo. Then LF1 stores the keywords with storage information of the photo in the ElasticSearch for furture indexing. When user types in a sentence like "show me my cats", it is handled by LF2, which further utilize Lex to parse the sentence and get the keyword ('cats' in this case), then return the corresponding photos that contains that keyword.




# Usage
A cloudformation template yaml file is in the root directory, which automatically builds up all the necessary Amazon Web Services, including two pipelines that help automatic code update from github and continuous delivery.

# LFPipeline:
Build two lambda functions and the S3 storing photos (there's a trigger so I put them together for convenience). Plus, api gateway is also built here. PipelineS3: This is for the front-end code. Note that the front-end code is in the front-end branch

# Elastic Search
Note that there's no template for Lex so it should be built manually.

# 1. Launch an ElasticSearch instance1
a. Using AWS ElasticSearch service2, create a new domain called “photos”. b. Make note of the Security Group (SG1) you attach to the domain. c. Deploy the service inside a VPC3. i. This prevents unauthorized internet access to your service.

# 2. Upload & index photos
Create a S3 bucket (B2) to store the photos.
Create a Lambda function (LF1) called “index-photos”. i. Launch the Lambda function inside the same VPC as ElasticSearch. This ensures that the function can reach the ElasticSearch instance. ii. Make sure the Lambda has the same Security Group (SG1) as ElasticSearch.
Set up a PUT event trigger4 on the photos S3 bucket (B2), such that whenever a photo gets uploaded to the bucket, it triggers the Lambda function (LF1) to index it. i. To test this functionality, upload a file to the photos S3 bucket (B2) and check the logs of the indexing Lambda function (LF1) to see if it got invoked. If it did, your setup is complete. ● If the Lambda (LF1) did not get invoked, check to see if you set up the correct permissions5 for S3 to invoke your Lambda function.
Implement the indexing Lambda function (LF1): i. Given a S3 PUT event (E1) detect labels in the image, using Rekognition6 (“detectLabels” method). ii. Use the S3 SDK’s headObject method7 to retrieve the S3 metadata created at the object’s upload time. Retrieve the x-amz-meta-customLabelsmetadata field, if applicable, and create a JSON array (A1) with the labels. iii. Store a JSON object in an ElasticSearch index (“photos”) that references the S3 object from the PUT event (E1) and append string labels to the labels array (A1), one for each label detected by Rekognition. Use the following schema for the JSON object: { “objectKey”: “my-photo.jpg”, “bucket”: “my-photo-bucket”, “createdTimestamp”: “2018-11-05T12:40:02”, “labels”: [ “person”, “dog”, “ball”, “park” ] }
# 3. Search
a. Create a Lambda function (LF2) called “search-photos”. i. Launch the Lambda function inside the same VPC as ElasticSearch. This ensures that the function can reach the ElasticSearch instance. ii. Make sure the Lambda has the same Security Group (SG1) as ElasticSearch. b. Create an Amazon Lex bot to handle search queries. i. Create one intent named “SearchIntent”. ii. Add training utterances to the intent, such that the bot can pick up both keyword searches (“trees”, “birds”), as well as sentence searches (“show me trees”, “show me photos with trees and birds in them”). ● You should be able to handle at least one or two keywords per query. c. Implement the Search Lambda function (LF2): i. Given a search query “q”, disambiguate the query using the Amazon Lex bot. ii. If the Lex disambiguation request yields any keywords (K, …, K), 1 n search the “photos” ElasticSearch index for results, and return them accordingly (as per the API spec). ● You should look for ElasticSearch SDK libraries to perform the search. iii. Otherwise, return an empty array of results (as per the API spec).

# 4. Build the API layer
a. Build an API using API Gateway. b. The API should have two methods: i. PUT /photos Set up the method as an Amazon S3 Proxy8. This will allow API Gateway to forward your PUT request directly to S3. ● Use a custom x-amz-meta-customLabelsHTTP header to include any custom labels the user specifies at upload time. ii. GET /search?q={query text} Connect this method to the search Lambda function (LF2). c. Setup an API key for your two API methods. d. Deploy the API. e. Generate a SDK for the API (SDK1).

# 5. Frontend
a. Build a simple frontend application that allows users to: i. Make search requests to the GET /search endpoint ii. Display the results (photos) resulting from the query iii. Upload new photos using the PUT /photos ● In the upload form, allow the user to specify one or more custom labels, that will be appended to the list of labels detected automatically by Rekognition (see 2.d.iii above). These custom labels should be converted to a comma-separated list and uploaded as part of the S3 object’s metadata9 using a x-amz-meta-customLabels metadata HTTP header. For instance, if you specify two custom labels at upload time, “Sam” and “Sally”, the metadata HTTP header should look like: ‘x-amz-meta-customLabels’: ‘Sam, Sally’ b. Create a S3 bucket for your frontend (B1). c. Set up the bucket for static website hosting (same as HW1). d. Upload the frontend files to the bucket (B2). e. Integrate the API Gateway-generated SDK (SDK1) into the frontend, to connect your API.

# 6. Deploy your code using AWS CodePipeline12
a. Define a pipeline (P1) in AWS CodePipeline that builds and deploys the code for/to all your Lambda functions b. Define a pipeline (P2) in AWS CodePipeline that builds and deploys your frontend code to its corresponding S3 bucket 8. Create a AWS CloudFormation13 template for the stack a. Create a CloudFormation template (T1) to represent all the infrastructure resources (ex. Lambdas, ElasticSearch, API Gateway, CodePipeline, etc.) and permissions (IAM policies, roles, etc.). At this point you should be able to:

1. Visit your photo album application using the S3 hosted URL.
2. Search photos using natural language via voice and text.
3. See relevant results (ex. If you searched for a cat, you should be able to see photos with cats in them) based on what you searched.
4. Upload new photos (with or without custom labels) and see them appear in the search results.
