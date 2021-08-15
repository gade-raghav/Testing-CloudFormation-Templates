# Testing-CloudFormation-Templates

## Project Description

This project aims to implement a solution that can optimize the run/test for CF template. We will be running a cloudformation template and testing it without actually launching the resources. We will be using [EKS](https://github.com/aws-quickstart/quickstart-amazon-eks) open-source cloudformation template for demonstration. We will be testing and stimulating cloudformation templates with zero cost.

### The project has been divided into 3 steps:
#### Step-1: Setup and Pre-requisites
#### Step-2: Static Code Analysis 
#### Step-3: Unit-Testing


## Setup and Pre-requisites

We will be using [EKS Control Plane template](https://github.com/aws-quickstart/quickstart-amazon-eks/blob/main/templates/amazon-eks-controlplane.template.yaml).

```
mkdir templates

touch amazon-eks-controlplane.template.yaml

```
(Copy pasted the contents of that template in this file.)

**Note**: I've checked into the official documentation and made some changes in the template accordingly as changes were required in  Resource properties.


Tools that we will be using are pip installable.

Pre-requisites:
- Python3.9
- [Cloud-Radar](https://github.com/DontShaveTheYak/cloud-radar)
- Pre-commit
- GIT

**Note**: This is run and tested on Operating System: Ubuntu 18.04 LTE

We will first set up a virtual environment for Python 3.9.

```
python3.9 -m venv env

source env/bin/activate 

```

The first pip dependency we will be installing is **pre-commit**. It helps us setting up pre-requisites that have to be met before committing the code (We will be using git version control system.

`pip install pre-commit`

Use `pre-commit -V` to check if it's working. We will be storing our dependencies in the requirements.txt file.

`pip freeze > requirements.txt`

Let's set up pre-commit with some checks. We will create **.pre-commit-config.yaml** file and add some checks.

`touch .pre-commit-config.yaml`

Add these lines to the files. We are using two basic hooks, end-of-file-fixer and trailing-whitespaces. Hooks are of two types, client-side and server-side. Client-side hooks are triggered by operations such as committing and merging (which we will be using), while server-side hooks are run on receiving pushed commits.

```
repos:
-   repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v2.3.0
    hooks:
    -   id: end-of-file-fixer
    -   id: trailing-whitespace
```    

We'll now run pre-commit.

`pre-commit install`

Now we will stage our files and commit them and we can see how pre-commit works when we commit our files.

```
git add .
git commit -m "Added pre-commit"

```
We will see that the checks have passed and a commit has been successful.

Our development environment is now set up. In our next step, we will do static code analysis.

## Static Code Analysis

Static code analysis is testing the code without actually running it. There are multiple types of static code analysis but we will be using Linters and Static Application Security Test. SAST analyzes code to find security vulnerabilities. A linter does the following:  flag programming errors, bugs, stylistic errors, and suspicious constructs.

### Linting

We will be using [cfn-lint](https://github.com/aws-cloudformation/cfn-lint). It validates AWS CloudFormation json/yaml templates against the AWS CloudFormation Resource Specification. Documentation and the rules can be found in the aforementioned link.
With pre-commit installed it's easy to add new checks. To add cfn-lint, we need to modify .pre-commit-config.yaml file. We should use the latest version of cfn-lint to not get unnecessary errors. **files** consists of a template directory which will consist of the CloudFormation template that we are testing.

```
repos:
-   repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v2.3.0
    hooks:
    -   id: end-of-file-fixer
    -   id: trailing-whitespace
-   repo: https://github.com/aws-cloudformation/cfn-python-lint
    rev: v0.53.0  # Latest cfn-lint version
    hooks:
    -   id: cfn-python-lint
        files: templates/.*\.(json|yml|yaml)$

```

We can test it using the following command.

`pre-commit run cfn-python-lint --all-files`

Now that it's working, let's add the changes and commit the file.

```
git add .

git commit -m "Added cfn-lint hook"

```

When we will pass the commit command, cfn-lint will be triggered and it will run its checks on our template. Once these tests are passed, our commit will be successful.

### SAST (Static Application Security Test)

We will be using [cfn-nag](https://github.com/stelligent/cfn_nag) tool to scan our template for potential security risks.

We will be adding the cfn-nag check to our pre-commit config file. We will be using our local hook. The following lines need to be added in .pre-commit-connfig.yaml file. **docker_image hooks** can be conveniently configured as local hooks. The entry specifies the docker tag to use. If an image has an ENTRYPOINT defined, nothing special is needed to hook up the executable.

Our .pre-commit-config.yaml file will look like this after making the changes.

```
repos:
-   repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v2.3.0
    hooks:
    -   id: end-of-file-fixer
    -   id: trailing-whitespace
-   repo: https://github.com/aws-cloudformation/cfn-python-lint
    rev: v0.53.0  # Latest cfn-lint version
    hooks:
    -   id: cfn-python-lint
        files: templates/.*\.(json|yml|yaml)$
-   repo: local
    hooks:
    -   id: cfn-nag
        name: cfn-nag
        language: docker_image
        entry: alpine/cfn-nag:latest --input-path /src
        files: templates/
        pass_filenames: false
```

We will test cfn-nag using the following command.

`pre-commit run cfn-nag --all-files`

Now that it's working, let's add the changes and commit the file.

```
git add .

git commit -m "Added cfn-nag hook"

```

In the next step, we will do Unit Tests so that we can test the template locally without worrying about AWS credentials.


## Unit Tests

A unit test is a way of testing a unit - the smallest piece of code that can be logically isolated in a system. In most programming languages, that is a function, a subroutine, a method, or property. Our template will be our application and all the AWS resources and conditions will be the units.

We will be unit testing our template using [Cloud-Radar](https://github.com/DontShaveTheYak/cloud-radar).
Cloud-Radar is a python module that allows testing of Cloudformation Templates/Stacks using Python. We don't have to deal with the AWS credentials and we don't have to deploy the resources to test the template.

##### Setup

We will be installing pytest and cloud-radar (version 0.6.0 as it is stable).

`pip install pytest`

`pip install cloud-radar==0.6.0`

Let's update our requirements.txt file.

`pip freeze > requirements.txt`

We will create the following directory structure to hold our test.
```
mkdir -p tests/unit

touch tests/unit/test_eks_controlpane.py
```

##### Writing Tests

**cloud-radar** works by reading our Cloudformation template and then rendering it the same way that the AWS Cloudformation service would. Our template consists of multiple parameters and resources that are to be created. We will be testing all of the resources that are going to be created.

Let's start writing our tests. We will first import pytest and fetch our template path in order to use the template. The following is our code to achieve the same.

```
from pathlib import Path

import pytest

from cloud_radar.cf.unit import Template


@pytest.fixture(scope='session')
def template_path() -> Path:
    base_path = Path(__file__).parent
    template_path = base_path / Path('../../templates/log-bucket.template.yaml')
    return template_path.resolve()


```

Fixtures are functions, which will run before each test function to which it is applied. Fixtures are used to feed some data to the tests and here it will provide a template path.


Let's write our test for the template. Create a Template object using the path to a Cloudformation template. We will create a dictionary of parameters and pass it to the template.

```
def test_ephemeral_bucket(template_path: Path):

    # Create a Template object using the path
    # to a Cloudformation template.
    template = Template.from_yaml(template_path)

    region = "us-west-2"

    SecurityGroupIds = ['sg-6979fe18','sg-6979fg21'] #Example Security group ids used
    SubnetIds = ['subnet-6782e71w','subnet-6792e32e'] #Example subnet ids used 
    KubernetesVersion = '1.14'
    RoleArn = 'arn:aws:iam::555555555555:role/eks-service-role-AWSServiceRoleForAmazonEKS-EXAMPLEBQ4PI' # Example Iam role arn used
    Ipv4Cidrs = ['10.100.0.0/16'] # Example Ipv4 Cidrs used
    EKSEncryptSecrets = 'Enabled'
    IamOidcProvider = 'Enabled'
    EKSClusterName = "cf-testing"

    # Create a dictionary of parameters for our template.
    params = {"EKSClusterName": EKSClusterName ,"SecurityGroupIds":SecurityGroupIds, "SubnetIds": SubnetIds,"RoleArn":RoleArn, "KubernetesVersion": KubernetesVersion , "Ipv4Cidrs": Ipv4Cidrs, "EKSEncryptSecrets": EKSEncryptSecrets, "IamOidcProvider": IamOidcProvider }

    # Render the template using our parameters and region.
    result = template.render(params, region)

```

The return of the template.render() is a dictionary that has all the CloudFormation functions and conditions solved.

We can print our template using the following piece of code.

```
import json
print(json.dumps(result, indent=4, default=str))

```


We will first make sure that proper resources have been created.

```
    # Check if proper resources have been created 
    resource_list = ["KMSKey","EKS","CleanupLoadBalancers","CallerArn", "ClusterOIDCProvider",]
    for resource in resource_list:
        assert resource in result["Resources"]

```

After we make sure proper resources are created, we will check each one of the resources by mapping the parameters and checking the conditions that were passed.

```
    # Test KMS Policy
    KMSKey_Policy = result["Resources"]['KMSKey']['Properties']['KeyPolicy']
    statement = KMSKey_Policy['Statement'][0]
    assert template.AccountId in statement['Principal']['AWS']

    # Test CleanupLoadBalancers
    CleanupLoadBalancer_resource = result["Resources"]['CleanupLoadBalancers']['Properties']
    assert template.AccountId in CleanupLoadBalancer_resource['ServiceToken']
    assert  "cf-testing" == CleanupLoadBalancer_resource['ClusterName']

    # Test CallerArn
    CallerArn_resource = result["Resources"]['CallerArn']['Properties']
    assert template.AccountId in CallerArn_resource['ServiceToken']

    # Test EKS
    EKS_resource = result["Resources"]['EKS']['Properties']
    assert "cf-testing" == EKS_resource['Name']
    assert "1.14" == EKS_resource['Version']
    for id in ['sg-6979fe18','sg-6979fg21']:
        assert id in EKS_resource['ResourcesVpcConfig']['SecurityGroupIds']

    for subnet in ['subnet-6782e71w','subnet-6792e32e']:
        assert subnet in EKS_resource['ResourcesVpcConfig']['SubnetIds']

    assert template.AccountId in EKS_resource['RoleArn']
 
```

**Note**: AccoutId is default to "555555555555" and other default values are listed [here](https://github.com/DontShaveTheYak/cloud-radar#usage).

We can also check the outputs section.

```
# Test Outputs

outputs = result['Outputs']

assert "cf-testing" == outputs['EKSName']['Export']['Name']
assert template.AccountId in outputs['EksArn']['Export']['Name']

```

The unit tests are working. Let's update .pre-commit-config.yaml file to run them for every commit. We will add pytest hook to the config file. After making the changes, the config file will look as below.

```

repos:
-   repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v2.3.0
    hooks:
    -   id: end-of-file-fixer
    -   id: trailing-whitespace
-   repo: https://github.com/aws-cloudformation/cfn-python-lint
    rev: v0.53.0  # Latest cfn-lint version
    hooks:
    -   id: cfn-python-lint
        files: templates/.*\.(json|yml|yaml)$
-   repo: local
    hooks:
    -   id: cfn-nag
        name: cfn-nag
        language: docker_image
        entry: alpine/cfn-nag:latest --input-path /src
        files: templates/
        pass_filenames: false
    -   id: pytest
        name: pytest
        entry: pytest
        language: system
        pass_filenames: false
        types_or: [python, yaml]

```

Let's test pre-commit

`pre-commit run --all-files`

Now we'll add and commit the changes.

`git add .`

`git commit -m "Added Unit Test"`



We can now successfully test CloudFormation Template Without deploying to AWS.
