
# Project Intent

This project, `CodePipeline Versioner` enables projects to internally version themselves in the CodePipeline build process within AWS. Projects should keep track of their major/minor version (1.0, 1.1, 1.2) but keeping track of build numbers is cumbersome. This AWS Lambda function enables projects to automatically version the build numbers simply by invoking the function. 

The function can be invoked a number of ways, for instance via Bash: 

```bash
```