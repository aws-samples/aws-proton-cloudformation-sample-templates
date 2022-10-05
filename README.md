## AWS Proton Sample CloudFormation Templates
This repository is a curated list of sample templates to use within AWS Proton that are authored for integration with AWS CloudFormation.

To use this repository, browse to the folder that corresponds to the template that you want to use. You will find there:
- An architecture description of the template
- A list of all the parameters required for the template
- The full content of a template, ready to be registered into AWS Proton
- A `spec` directory with an example spec file `spec.yaml` that you can use to create an
environment or a service from the example template.
- A link to a repository with basic code that runs on each of the templates, in case you want to fork it to use it as the basis for your deployment. The basic code is hosted in the [AWS Proton sample services](https://github.com/aws-samples/aws-proton-sample-services) repository

If you are looking for sample templates to use Terraform go to our [AWS Proton Terraform sample templates](https://github.com/aws-samples/aws-proton-terraform-sample-templates) library


## Registering Templates Using Template Sync
All of the Templates in this directory are set up to work with AWS Proton Template Sync. This repository is also a Github Template Repository. So you can click "Use this template" on the home page of this repo and that will create an identical repo in your account, which you can then use with template sync.

## Security
See CONTRIBUTING for more information.

## License
This library is licensed under the MIT-0 License. See the LICENSE file.
