
# Monkton CloudFormation Templates

This repo has CloudFormation tools that we leverage in our CI/CD pipelines with AWS CodePipeline. 

* [CloudFormation Custom Resource Lookup](lookup/)
* [CodePipeline Build Versioner](versioner/) (In Progress)

# Using these Templates

These aren't always our production templates. Please use with caution and vet internally before deploying. We avoid using "*" resources for things as best practice because—thats bad. These templates attempt to follow best practices at all times. 