### AWS XRay

#### What is it?

* Collects data about the *requests* that your AWS application serves, and provides tool that you can use to view, filter and get insights to identify issues and opportunities to optimise. e.g. It gives you a way to instrument requests coming through a collection of AWS services (APIGW, Lambda, Dynamo etc.)

#### How does it work?

* AWS provide an SDK which you can use to add tracing to:
  * The node HTTP module (therefore instrumenting all http requests in node)
  * The entire node AWS SDK, or individual services within it (S3, Dynamo etc.)
  * Any function, as you can create a new custom Trace object in Javascript that could be running inside a Lambda, ECS or EC2 instance.

In addition, XRay integrates directly with AWS services such as Lambda, API GW, Appsync, ELB etc. These services can be configured to either create traces or pass through traces from upstream services. In theory this allows for the ability to trace an application through the entire request/response cycle, or from one end of a pipeline to another. 

In order to trace a serverless application that accepts requests from API Gateway and processes them with Lambda it is as simple as turning XRay on in your API and in the Lambda. When requests flow through your system you can see a map of the services the request has been through, along with useful telemetry such as execution time, memory usage etc. 

#### Caveats

This sounds great, but there is one big caveat. XRay is not supported in the way you'd expect by all AWS services, most notably SQS. In our team we have a number of "pipeline" systems of lambdas connected with SNS and SQS. Lambdas can create or pass through traces to other services, SNS can pass through traces to Lambdas that have tracing enabled, or to SQS. The big problem is that SQS currently cannot pass through traces to Lambda. Given APIGW->Lambda->SNS->SQS->Lambda->DynamoDB, the trace id generated by APIGW will only be passed as far as SQS, which cannot automatically (read on for more on this) pass the trace to the Lambda. In practice this means that the Lambda connected to SQS will create its own trace id which will be passed through to dynamo. In XRay in the console the service graph will consist of two clusters, one APIGW->Lambda->SNS->SQS and one Lambda->DynamoDB. 

It is possible to get this scenario to work, but not without some manual intervention. You need to turn off dynamic tracing on the lambda receiving the message. Then in the function itself find the trace id (which is an attribute of the message), then use the XRay API to create a new Segment with the trace id and send it to services downstream. 

This issue is being worked on, but no ETA has yet been given 

* https://github.com/aws/aws-xray-sdk-java/issues/246
* https://github.com/aws/aws-xray-sdk-node/issues/208

#### Datadog integration

The integration with datadog appears to be very rich and you can collect logs and traces in the same place. It also creates dashboards automatically based on the xray service graphs. Collecting logs and metrics in the same place looks like it requires specific tagging for the logs and telemetry that need to be explored in greater depth. 