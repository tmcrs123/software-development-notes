
## Files

`terraform.tfstate` - this file is essentially a list of all or resources. Internally this is the file Terraform uses to keep track of all resources and theirs ids. This is sensitive information and needs to have restricted access 

`resource "aws_iam_role" (type) "lambda_exec_role" (name)` 

dotnet lambda package --configuration Release --output-package ./publish/lambda.zip
## Commands

`terraform init` - initialize the project and download plugins
`terraform apply` -  provision the resources, use `-var "{variable_name}={variable_value}"` to override a variable defined elsewhere
`terraform destroy` - delete
`terraform fmt` - format the configurations in the current file
`terraform validate` - validate the configuration file
`terraform show` - list all resources
`terraform state` - manually manage state (advanced) 
`terraform output` - list all the outputs defined
`terraform workspace select {workspace name}` - change to an existing workspace
`terraform workspace new {workspace name}` - create a new workspace
