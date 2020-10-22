# Cloudformation; preventing accidental deletions

Infrastructure as code has been a game changer for software engineers. Being able to spin up whole service architectures of compute, storage and networking in a quick, convenient, testable and reproducible way increases developer confidence and reduces risk.

However, with the speed and ease of creation also comes the speed and ease of destruction, and when we’re talking about databases this can lead to big problems.

> I just deleted the database on production...

If you use AWS Cloudformation there are some template attributes you can use to prevent deletion of resources. 

### DeletionPolicy

This attribute applies to all types of Cloudformation resources and is used to tell Cloudformation what to do if instructed to delete the resource. 

| Attribute Value          | Effect                                                       |
| ------------------------ | ------------------------------------------------------------ |
| _No attribute specified_ | Cloudformation will delete the resource                      |
| `Retain`                 | Cloudformation will not delete the resource, but will remove the resource from the stack |
| `Snapshot`               | Cloudformation will take a snapshot backup of the resource and retain it. Then delete the original resource. |

_Note: Only some resource types support snapshots, and there are some exceptions to the default behaviour where no `DeletionPolicy` attribute is specified. See the [AWS Documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html) for more details._

### Stack Policy

A stack policy is set on the stack itself and consists of a series of statements that define the operations that can be performed on specific resources or resource types within the stack. Once a stack policy is set on a stack, all resources are protected from any updates or deletions. The policy you create defines which resources or resource types are permitted to have update or delete actions performed on them. 

For example, you could set a policy to prevent deletion of a specific database resource like this:

```json
{
  "Statement" : [
    // explicitly allow updates of any other resource in the stack
    {
      "Effect" : "Allow",
      "Action" : "Update:*",
      "Principal": "*",
      "Resource" : "*"
    },
    // explicitly deny deletes of a specific resource in the stack
    {
      "Effect" : "Deny",
      "Action" : "Update:Delete",
      "Principal": "*",
      "Resource" : "ResourceId/SomeDatabase"
    }
  ]
}
```

or set a policy to prevent the deletion of specific resource types in the stack like this:

```json
{
  "Statement" : [
    // explicitly allow updates of any other resource in the stack
    {
      "Effect" : "Allow",
      "Action" : "Update:*",
      "Principal": "*",
      "Resource" : "*"
    },
    // explicitly deny deletes of all RDS instances
    {
      "Effect" : "Deny",
      "Action" : "Update:Delete",
      "Principal": "*",
      "Resource" : "*",
      "Condition" : {
      	"StringEquals" : {
        	"ResourceType" : ["AWS::RDS::DBInstance"]
      	}
    }
  ]
}
```

If you're using native Cloudformation then a stack policy is defined as a separate JSON document and applied using the AWS CLI. We use [serverless framework](https://serverless.com), which allows for the policy document to be embedded in the serverless configuration itself like this:

```yaml
provider:
  name: aws
  runtime: nodejs12.x
  stackPolicy:
    - Effect: Deny
      Action: "Update:Delete"
      Principal: "*"
      Resource: "*"
      Condition:
        StringEquals:
          ResourceType:
            - AWS::RDS::DBCluster
    - Effect: Allow
      Action: "Update:*"
      Principal: "*"
      Resource: "*"
resources:
  Resources:
    IonBridgeTransformerQueue:
      Type: "AWS::RDS::DBCluster"
      Properties:
       ...
```

### Stack Termination Protection

This configuration prevents the entire stack from being deleted. It does not protect individual resources from being deleted and can only be enabled via the cli or console. This is very easy to configure but felt like a blunt instrument for our needs. 

### IAM policy

This method uses IAM to restrict the ability to make specific Cloudformation API calls on a stack. You can use this to restrict the IAM user or role from deleting the whole stack by applying a policy like this:

```json
{
    "Statement":[{
        "Effect":"Deny",
        "Action":[
            "cloudformation:DeleteStack",
            "cloudformation:UpdateStack"
        ],
        "Resource":"arn:aws:cloudformation:eu-west-1:123456789012:stack/SomeStack/*"
    }]
}
```

or specific resource types within the stack with a policy like this:

```json
{ 
  "Statement": [ 
    { 
      "Effect": "Deny", 
      "Action":[
        "cloudformation:DeleteStack",
        "cloudformation:UpdateStack"
      ],
      "Resource": "*", 
      "Condition": { 
        "StringEquals": {
          "cloudformation:ResourceTypes": [ 
            "AWS::RDS::DBCluster" 
          ] 
        } 
      } 
    } 
  ] 
}
```



## So which one should I use?

As always "it depends". In this case on what behaviour you want to achieve and your particular AWS setup.

When researching the preferred approach I had the following outcomes in mind:

* Accidental deletions of databases (or other critical resources) would not be applied, and result in a red builds
* Intentional deletions of databases are possible, but there should be extra care and accountability applied
* Even when database deletions are intentional, have the data backed up as an extra precaution

Stack Termination protection was ruled out because it can only be enabled/disabled in the AWS CLI or the console and therefore gives no audit trail for changes to the settings. Also it operates at the stack level, not the resources within the stack. 

The `DeletionPolicy` attribute is great for providing a backup of the resource to be deleted, but because it will either delete the resource, or remove it from the stack, it will not result in a build failure.  

Furthermore, if the `DeletionPolicy` attribute it set to `Snapshot`, in the case of RDS, a snapshot backup must be restored to a brand new cluster which will be outside of the Cloudformation stack. If the `DeletionPolicy` attribute is set to `Retain` then the database will be kept, but orphaned from the Cloudformation stack.

Either way any subsequent deployments of our current Cloudformation template would fail because we'd be trying to create a resource that already existed outside of the stack. The way to solve this is to import the resource back into the stack and the effort to do this isn't worth it for our needs.

The IAM policy route would give us the red build if we tried to delete a resource we shouldn't. It has the benefit of being able to apply a more global _no-delete_ policy for specific resource or groups, but this didn’t fit our needs due to the way that IAM roles for deployment are managed.  

We decided the best approach for our needs was stack policies. Creating a policy that applied only to the resources we wanted to protect was simple. It had the desired effect of a red build if attempting to delete a protected resource. The policy is stored right in the serverless template so it’s easy to find, and is under source control so it can be changed if desired and gives us good visibility/accountability for changes. 

### Side Note

When resarching for the best approach here I began to think about what _other_ things we could do to defend against accidental deletions. 

* Is your database the primary source of truth? Can you use cheap storage options to persist records and then load them into a database for querying in a normalised way? This forces you to think about how the data is loaded into the database in the first place and allows you to more easily create mechansims to repopulate the database in case of disaster. 
* Can you structure your deployables in a more risk averse way? If you have a Cloudformation stack containing compute resources running application code and database resources, would it be better to create two stacks and deploy them separately. The application code is likely to change at a much higher cadence than the database configuration. Having the two separated may lead to less accidental changes.

Dont let this sort of thing be an afterthought. If you have non-production environments to test with then try deleting important infrastructure, see what happens and how you might respond to it. You will learn what could go wrong and be able to put a recovery plan in place, or make changes now to mitigate the risks you've identified. Doing this will increase your team's confidence and knowledge of the applications they're building.