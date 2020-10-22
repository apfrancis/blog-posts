---

The Acuris Technology Stack
Over the past number of years the Acuris technical landscape has changed hugely. We're becoming less reliant on our legacy systems, we've embraced a microservices architecture running in the cloud and there's a feeling amongst our engineering teams that we're building software 'the right way'.
In this post we'll talk about our current tech stack and how we see it evolving.
We have content teams around the world creating content using our in-house tools. On the distribution side we have services supplying specialist news, research and analysis to financial professionals across the globe via websites, apps and emails.


---

Our Philosophy
Our engineering teams have autonomy to make their own technology choices in order to build and support their respective products. As a larger engineering discipline we have agreed on a general set of tools and technologies we use in order to meet our business goals.
These technologies have been 'battle tested' in the organisation and represent the result of a number of iterations of both software and platform development. 
We approach the use of new technologies with cautious optimism. The technology landscape never stands still, but moving forward must be balanced with the effort and costs of doing so. 
Introducing new languages and frameworks requires investment to skill our engineers and define standards and tooling. Supporting multiple languages and tools that accomplish the same task increases the maintenance and operational costs. It also increases the number of attack vectors for malicious code. 
For these reasons we tend to introduce new technologies when there is a valid reason to do so, not just because they are new and sexy. 


---

The Stack
Infrastructure
We use Amazon Web Services for almost all of our infrastructure, including our customer-facing and back office applications, APIs, databases, and CI tooling.
Our microservices are packaged into Docker containers and are deployed to Amazon ECS inside of our VPC. Traffic is routed to the services using Amazon Load Balancers. 
We use Fastly as our content delivery network, which ensures that our content is delivered from edge nodes that are geographically closest to our users. 
Databases
Depending on the use case we have a number of 'go to' database technologies. We do not host these ourselves, preferring a buy over build strategy and offloading the maintenance and setup effort to third party providers. 
Mongodb or Dynamodb - for general purpose document storage. We've had mixed results with Mongo and are now looking to move to alternatives such as DynamoDb.
Elasticsearch - for free text and faceted searching. Also used to match users to our content in our notifications engine. 
Postgres - for relational data where we can't easily predict an access pattern. Also being used as a document store using JSON types.
We also have a number of systems that rely on a legacy MSSQL database. We have adopted the strangler pattern in order to gradually migrate all of our data into other databases. 
Messaging
A number of our service architectures feature message queues for redundancy and resilience. RabbitMQ has long been our MQ of choice and we have a well established in-house library for Node and Go. Recently we've experimenting using SNS/SQS in some teams in order to reduce the operational overhead of managing Rabbit clusters. SNS/SQS is less feature rich, but some of the features that our services use in Rabbit could be easily written in code. 
In addition to operational overhead, our hosted Rabbit clusters are charged monthly with a cap on messages per cluster. SNS/SQS is charged per message and would result in significant cost savings. 
Programming Languages and Frameworks
Some years ago the number of languages used within the engineering team was out of control. These days our services are written largely in Javascript/NodeJS or Go. We chose the language depending on the individual use case, with Go preferred when we know we need CPU or memory performance. 
Most teams are using (and loving) Typescript, which offers a really great developer experience when coupled with a capable IDE like Visual Studio Code. 
We have a number of proud Gophers who are fostering the learning and development of Go across our Engineering teams. 
In order to reduce the technical footprint further we've started to create in-house libraries to wrap common features such as logging and http requests. 
We've started to assemble an awesome-lists style repo for all the libraries, tools and APIs in the business. 
Frontend
Our frontend is written entirely in React, and we have a large (and growing) set of shared UI components using a Monorepo managed by Lerna. 
The shared components are written in Typescript and use the Styled Components library to reduce the complexity involved in having large shared CSS files. 
Components can be viewed in a "living style guide" generated using React Styleguidist. This allows product owners and designers to understand the capabilities of each component as they are defining their requirements. 
Our client-facing sites are designed in a mobile-first manner so that they look good on smaller screens. We have a mobile app available on iOS built in React Native.
Logging and Telemetry
We have centralised our logging and telemetry UI within Datadog. Teams are empowered to create their own monitors, dashboards and alert mechanisms for the services they build and support. 
Each application running in ECS emits logs to stdout/stderr which are shipped to Datadog via the Datadog Agent. Metrics are emitted via the StatsD protocol and aggregated using DogstatsD and pushed into Datadog at intervals. 
For transaction checking we are experimenting with Datadog Synthetics 
CI and Build Tools
Most of our infrastructure in AWS is creating using Terraform, which allows us to keep our configuration under source control and have exactly reproducible builds.
We terraform as much as we can, even down to our Datadog dashboards! We have built a number of boilerplate configurations to be able to provision infrastructure for common purposes. 
We have in-house tooling that wraps terraform in order to build and deploy services onto AWS. These deployments are managed with Jenkins where pipelines are configured to build, test and deploy. 
Many teams now practice continuous deployment, whereby once their code has been merged to master it will go live if the tests pass. New features are hidden behind feature flags. We currently have in-house tech for managing feature flags but have recently been looking at Unleash to do this for us. 
Testing
Testing is one area of technology where we haven't yet settled on common tooling. Our services use a plethora of libraries. 
Javascript Testing Libraries - Jest, Ava, Mocha (with a sprinkling of assertion libraries)
Acceptance Test Frameworks - Rspec, CucumberJS, TestCafe, Nightwatch
Load testing tools - Gatling, K2
APIs
At Acuris we have both internal and external (client facing) APIs. We are currently experimenting with JSON schema to define the responses from our APIs and generating those from the type definitions in our projects. 
Our internal APIs are only available inside the Amazon VPC to our internal services. 
External APIs use the Amazon API Gateway to feed requests to Lambdas which call services to resolve the request. As these APIs are external they are secured and only available to authorised users. 
We have experimented with using Swagger to better document our APIs. This is particularly important for external APIs that are to be consumed by engineers outside Acuris as it provides a living document based on the API configuration and doesn't require additional documentation to be kept up to date. 
Our internal APIs use Consumer Driven Contracts to ensure that teams don't deploy breaking changes to downstream consumers. We use Pact and Mockingjay Server for this
Conclusion
This is a very broad overview of the technologies we use at Acuris. Whilst we are happy with the current state of the stack we're always for ways to improve. Our experiments with Postgres, DynamoDb, Unleash, and JSON Schema are helping us to shape our technology landscape for the future.
In subsequent posts we'll take a deeper dive in to how these technologies help us deliver value to our customers. 
If you'd like to learn more about engineering at Acuris then feel free to comment or get in touch.