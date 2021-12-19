
# Project Intent

This project, `CloudFormation Lookup` enables you to build CloudFormation templates that pull configuration data from DynamoDB as a dictionary. Consider the scenario where you are using CodeBuild to deploy apps into a series of AWS accounts you control. Each of those accounts may have differening configuration data, depending on the intent of the account. For instance, perhaps you deploy an application across segregated tenents for customers? Each of those tenets may have different configurations, like DNS host names. 

As of right now with CloudFormation, there is no means to pull that data, on a per account basis. To solve this problem, we have developed a custom CloudFormation resource, that enables you to define resource, as below in your CloudFormation template: 

```yaml
# Looks up properties for automated deployments that we store within DynamoDB as configuration.
# The properties are looked up by key. The return values could be strings, string lists, 
# numbers, or maps. Maps enable you to put a bunch of values in the resulting data structure. 
PropertiesLookupTest:
  Type: Custom::PropertiesLookup
  Version: '1.0'
  Properties:
    # Identified the Lambda function which we will invoke for this custom resource
    ServiceToken: !Sub "arn:${AWS::Partition}:lambda:${AWS::Region}:${AWS::AccountId}:function:dsop-tools-lookup"
    # This is the lookup value that we will attempt to find from 
    # storage. We key this value on the name of the application PLUS
    # the account identifier. We may want to deploy everything with the same
    # configuration—or potentially have different tenets which have account
    # specific configurations. 
    Value: !Sub "/website/${AWS::AccountId}/${AWS::Region}"
    # This looks for the default value that maybe stored in the key. 
    # This is useful for auto deployed StackSet Instances in AWS. 
    DefaultLookupValue: "/website/default"
    # This default value, in JSON format if the value or default 
    # lookup are not able to be discerned
    DefaultValueAsJson: "{ \"domainName\": \"smoke-test.monkton.io\" }"
```

Once this has been defined in the `Resources` block of your CloudFormation template, you can reference the data anywhere in template. For instance, we may want to pass the `domainName` into a `ACM` certificate: 

```yaml
SampleCertificate: 
  Type: AWS::CertificateManager::Certificate
  Properties: 
    DomainName: !GetAtt PropertiesLookupTest.domainName
    ValidationMethod: DNS
```

Here, we first use the `PropertiesLookupTest` to lookup values based on the `Value` key. This is a key within the DynamoDB table that maps to the value you want. There are fallabck conditions as well. If the value cannot be found, it will look to the `DefaultLookupValue` value. If that isn't found, it will fallback to the `DefaultValueAsJson` value. 

This design enables you to create new environments that would have default values configured—but ones that could be customized on a per-environment basis. 

# To do

The last remaining task is to drop this Lambda function into a VPC. 

[_] Deploy into VPC

# What gets deployed

We have a series of objects that will be created. In the account that this project is deployed into, we create the following: 

* DynamoDB table to hold the configuration data
* KMS Key to protect the DynamoDB data
* IAM Role titles `master-dynamodb-account-role` that functions in the child accounts can assume
 * This has permissions to access the DynamoDB `GetItem` as well as the KMS `Decrypt` actions
* StackSet that deploys the StackSet Instances into the desired target

In each target account, we deploy: 

* The function to retrieve the data 
* IAM Role for the function to use to execute

*Why do we have to deploy a function to every account?* you intelligently ask? Because a single root Lambda is a total pain to try to work with. Deploying a Lambda into every account that pulls the data enables you to easily invoke it from CloudFormation. Simple as that. This design and assumption of roles enables you to pull data from a single root DynamoDB table with configuration data. 

# DynamoDB Data

There are limitations as of now. We limit the type of data to be returned to simple strings, string sets, or simple maps. String sets are joined into comma delimited strings (thus can be split in CloudFormation templates). Maps should only be a single level and contain only Strings and StringSets. 

The database should have these keys: 

* `settingKey`: this is the hash key in DynamoDB and must be set, this is what `Value` and `DefaultLookupValue` provide lookups for. 
* `storedValue`: this is the String, StringSet, or Map that has the values to be returned within it. 

# AWS Organizations Account Makeup

## Account Breakdown

This project deployment runs on the idea of account separation using *AWS Organiztions*. For instance, part of our account management and breakout is the following accounts: 

* DSOP Account
* N+1 Test Accounts
* N+1 Production Accounts

We then break those projects into the following OUs: 

* `DSOP OU` for our single DSOP account
* `Test OU` for our one to many test accounts
* `Production OU` for our one to many production accounts

Under our `Production OU` we also separate solutions into their own OUs. For instance, Monkton's Website would be under the `Monkton Website OU`, our Orchestrator app backend would be in the `Orchestrator OU`.

The structure looks like: 

* Root  
	* DSOP OU
		* DSOP Account
	* Test OU
		* Test Account 1
		* Test Account 2
	* Production OU
		* Monkton Website OU
			* Monkton Website App
		* Orchestrator OU
			* Shared Tenet App
			* USAF Tenet App
			* USSF Tenet App
			
Thus, when we apply this template to the `Production OU`, every account underneath it is now allowed to pull configuration data. 

When we use CodePipeline to update the Compute IaC for Orchestrator, we will apply it to the `Orchestrator OU` which will either create or update the Compute IaC StackSets for all the accounts in that OU. 

## Deployment 

To deploy this stack, we are going to use your `DSOP Account` as the *Delegated Admin* for deploying the `CloudFormation Lookup` StackSet. (You will need to delegate this account as a *Delegated Admin* via the root account CloudFormation settings). You will also need to do this if you are using your DevSecOps account and CodePipeline. 

Next, deploy the `dsop-parameter-lookup-stackset.yaml` template, providing the following parameters: 

* `parProjectDeploymentOU`: a path that leads to the production OU for your organization. It will follow this format: org/root/ou
* `parRegionDistribution`: a comma separated list of regions this could be deployed to. For instances `us-east-1,us-west-1`

The org OU structure can be determined by navigating to your Root account and your *AWS Organiztions* configuration. Tap on `AWS Accounts` and view the Organization structure. The path should look something like `o-XXXXXXXXXX/r-XXXX/ou-XXXX-XXXXXXXX`/ 

# Test StackSet

Our test StackSet relies on the same parameters and configuration. This enables you to configure a key in DynamoDB and it sends it to `Output` for the deployed StackSet Instances. 

