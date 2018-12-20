# concourse terraform

[concourse-ci](https://concourse-ci.org) tasks for running [terraform](https://www.terraform.io)

## table of contents

- [overview](#overview)

	- [features](#features)

	- [issues](#issues)

	- [usage](#usage)

		- [specifying the version](#specifying-the-version)

		- [providing input variable values](#providing-input-variable-values)

		- [configuring the backend](#configuring-the-backend)

		- [providing terraform source files](#providing-terraform-source-files)

		- [managing local state files](#managing-local-state-files)

	- [advanced usage](#advanced-usage)

		- [running `{tf-cmd}-consul` tasks with `consul-wrapper`](#running-tf-cmd-consul-tasks-with-consul-wrapper)

- [tasks](#tasks)

	- [plan](#planyaml-plan-with-no-output)

	- [apply](#applyyaml-apply-with-no-plan)

	- [create-plan](#create-planyaml-create-a-plan)

	- [show-plan](#show-planyaml-show-a-plan)

	- [apply-plan](#apply-planyaml-apply-a-plan)

- [development](#development)

- [helper scripts](#helper-scripts)

- [automated builds](#automated-builds)

# overview

this project provides a series of concourse-ci [tasks](https://concourse-ci.org/tasks.html) which are powered by a small python library that wraps and orchestrates terraform

## examples

see [examples](examples/README.md)

## features

- stays simple by minimizing management of remote or local state

	- remote state can be managed entirely by terraform using [backends](https://www.terraform.io/docs/backends/)

	- local state can be produced as an output and managed with concourse resources or scripts

- pre-built images for the terraform versions in `tf-versions` are available on docker hub, or you can build any version with the provided `Dockerfile`

- orchestrates the [plan and apply on different machines](https://www.terraform.io/guides/running-terraform-in-automation.html) automation strategy

	- generates plan archives using an absolute working dir `/tmp/tfwork/terraform`  
	(this ensures paths will be consistent when using plan files in separate pipeline steps).

	- plan archives can be persisted to remote storage using concourse resources (such as the [s3 resource](https://github.com/concourse/s3-resource))

## issues

- no explicit interface to `-var-file`, so var files can be provided via:

	- `.tfvars` files in the `terraform-source-dir`

	- with `TF_CLI_ARGS_{command}` parameters (see [environment variables](https://www.terraform.io/docs/configuration/environment-variables.html#tf_cli_args-and-tf_cli_args_name))

	- variables can still easily be provided with [task params](#providing-input-variable-values)

- [workspaces](https://www.terraform.io/docs/state/workspaces.html) are not yet supported

- most terraform commands other than those needed to provide a `plan -> approve -> apply` lifecycle are not yet supported

## usage

### specifying the version

the tasks require a pipeline to provide them with their docker image resource

this ensures you provide a specific version of the terraform image when executing

```yaml
resources:
# the terraform tasks
- name: concourse-terraform
  type: git
  source:
  	uri: https://github.com/Snapkitchen/concourse-terraform

# the terraform image
- name: concourse-terraform-image
  type: docker-image
  source:
    repository: snapkitchen/concourse-terraform
    tag: {VERSION}

jobs:
# a job using a terraform task with the terraform image
- name: terraform-task
  plan:
  - get: concourse-terraform
  - get: concourse-terraform-image
  - task: do-terraform-task
    image: concourse-terraform-image
    file: concourse-terraform/tasks/{TASK}.yml
```

you can also build the docker image yourself using the [docker-image-resource](https://github.com/concourse/docker-image-resource) and the `Dockerfile` from the source

### providing input variable values

tfvars can be provided by including `terraform.tfvars` or `*.auto.tfvars` files in the terraform source dir, or by passing them as parameters into the task

```yaml
jobs:
# a terraform task with variables
- name: terraform-task
  plan:
  - get: concourse-terraform
  - get: concourse-terraform-image
  - task: do-terraform-task
    image: concourse-terraform-image
    file: concourse-terraform/tasks/{TASK}.yml
    params:
      TF_VAR_my_var: my_value
```

### configuring the backend

the backend can be configured two ways:

- by including a `.tf` file containing the backend configuration in the terraform source dir

- by specifying the `TF_BACKEND_TYPE` parameter to automatically create a `backend.tf` file for you

the backend configuration can be dynamically provided with additional params in the form of `TF_BACKEND_CONFIG_{var_name}: {var_value}`

e.g. to automatically create and configure an s3 backend:

```yaml
jobs:
# a terraform task with variables
- name: terraform-task
  plan:
  - get: concourse-terraform
  - get: concourse-terraform-image
  - task: do-terraform-task
    image: concourse-terraform-image
    file: concourse-terraform/tasks/{TASK}.yml
    params:
      TF_BACKEND_TYPE: s3
      TF_BACKEND_CONFIG_bucket: my-bucket
      TF_BACKEND_CONFIG_key: path/to/my/key
      TF_BACKEND_CONFIG_region: us-east-1
```

results in `backend.tf`:

```hcl
terraform {
  backend "s3" {}
}
```

with init parameters:

```sh
terraform init \
	-backend-config="bucket=my-bucket" \
	-backend-config="key=path/to/my/key" \
	-backend-config="region=us-east-1"
```

### providing terraform source files

terraform source files are provided through the `terraform-source-dir` input and the `TF_WORKING_DIR` and `TF_DIR_PATH` parameters

by default, the `TF_WORKING_DIR` is used as both the working directory and the target for terraform, meaning terraform expects that all `.tf` files are contained in this directory

there are two ways to specify the terraform directory:

1. set `TF_WORKING_DIR` to the `terraform` dir path

	e.g. a resource `src` with the source tree containing:
	
	```
	src/terraform/terraform.tf
	```
	
	might configure the following task:
	
	```yaml
	- task: terraform-plan
	  image: concourse-terraform-image
	  file: concourse-terraform/tasks/plan.yaml
	  input_mapping:
	  	terraform-source-dir: src
	  params:
	  	TF_WORKING_DIR: terraform-source-dir/terraform
	```

	the advantage to the this method is that only the contents of the `src/terraform` are copied to the working directory. this may reduce the size of the plan artifact if you have a lot of extra files in your source tree.

2. set the `TF_DIR_PATH` to the `terraform` dir path relative to `TF_WORKING_DIR`

	e.g. a resource `src` with a source tree containing:
	
	```
	src/
	src/templates/example.tpl
	src/terraform/terraform.tf
	```

	with a terraform template that references a source tree path:
	
	```hcl
	data "template_file" "example" {
	  template = "${file("templates/example.tpl")}"
	}
	```
	
	might configure the following task:
	
	```yaml
	- task: terraform-plan
	  image: concourse-terraform-image
	  file: concourse-terraform/tasks/plan.yaml
	  input_mapping:
	  	terraform-source-dir: src
	  params:
	  	TF_DIR_PATH: terraform
	```

	the advantage to this method is that you can reference additional relative files outside of the terraform directory.

	this path would become invalid if the `TF_WORKING_DIR` was set to `terraform-source-dir/terraform`, since the working directory tree would contain
	
	```
	terraform.tf
	```

	where `templates/example.tpl` does not exist

	in that case setting `TF_DIR_PATH` to target the terraform dir inside the `TF_WORKING_DIR` would result in the working directory tree:

	```
	templates/example.tpl
	terraform/terraform.tf
	```
	
	with `terraform` being the target terraform directory

### managing local state files

**note**: this does not apply if using remote state backends

**warning**: if you configure the backend type as `local`, do not override the `path` [backend config](https://www.terraform.io/docs/backends/types/local.html), as changing the expected `terraform.tfstate` location may break orchestration and result in lost state files

#### saving local state

- the `terraform.tfstate` and `terraform.tfstate.backup` files will be made available in the `state-output-dir` after running `apply.yaml` or `apply-plan.yaml`

- you should configure an [ensure](https://concourse-ci.org/ensure-step-hook.html) task step hook to upload the state files to remote storage, which guarantees the state is persisted even if the apply fails

- **failure to do this may result in state files being lost when the container they were generated on is automatically destroyed**

#### loading local state

- when providing existing state to a task, you should configure the `STATE_FILE_PATH` parameter with the path to the input `terraform.tfstate` file

 this file will be copied to the `TF_WORKING_DIR` as `terraform.tfstate`

- if you need to provide the state from a concourse input, you can use the optional input `state-input-dir`

	e.g. for a state resource named `terraform-state`:

	```yaml
	- task: terraform-plan
	  image: concourse-terraform-image
	  file: concourse-terraform/tasks/plan.yaml
	  input_mapping:
	    state-input-dir: terraform-state
	  params:
	    STATE_FILE_PATH: terraform-state/terraform.tfstate
	```

## advanced usage

### running `{tf-cmd}-consul` tasks with `consul-wrapper`

#### using the pre-built image

a consul image is also provided and automatically built by docker hub

the image is available as per:

`snapkitchen/concourse-terraform:{tf-version}-consul`

the version of consul is determined by the value in `consul-version` at the time the image was built, thus it is recommended to build this image yourself if you want to use a specific consul version

#### configuring the `{tf-cmd}-consul` tasks

each terraform command task has a corresponding "with consul" version, named `{tf-cmd}-consul.yaml`

these tasks will run `dumb-init` as the entry point and then run the `consul-wrapper` script, which will:

- start dnsmasq, forwarding requests for `*.consul`
- start the consul agent
- join a cluster
- run the terraform phase
- on error, or success, leave the cluster and stop dnsmasq

they also include an additional optional input called `consul-certificates` which can be used to provide certificate files during authentication

#### configuring the consul agent

consul configuration must be provided through the [CONSUL_LOCAL_CONFIG](https://www.consul.io/docs/guides/consul-containers.html#configuration) environment variable, as the consul exe will be executed as `consul agent` with no additional parameters

to join a specific cluster host directly, set `CONCOURSE_TERRAFORM_CONSUL_JOIN` to the intended cluster host or ip used for `consul join`

otherwise, configure `retry_join` or similar to have the script wait for the leader status check to return OK

#### providing paths to resources for the consul provider

the most common use case is the need to provide ca certificates or client auth certificates and keys when authenticating to the consul cluster through the following environment variables:

- `CONSUL_CACERT`
- `CONSUL_CAPATH`
- `CONSUL_CLIENT_CERT`
- `CONSUL_CLIENT_KEY`

however, unless these paths are either already inside your `terraform-source-dir` (unlikely), or you override the `TF_WORKING_DIR` to include the entire concourse working directory, then these paths may become broken if you try to use `provider "consul" {}` in your terraform project

e.g. if your current working directory from concourse looks like this:

```
consul-certificates/
terraform-source-dir/
```

then any relative paths to the `consul-certificates/` folder will be broken if the `TF_WORKING_DIR` is set to the default of `terraform-source-dir` or anything deeper

to alleviate this issue, these additional environment variables can be provided to automatically set to the above variables to the absolute path of the relative file or directory path provided:

- `CONCOURSE_TERRAFORM_CONSUL_CACERT`
- `CONCOURSE_TERRAFORM_CONSUL_CAPATH`
- `CONCOURSE_TERRAFORM_CONSUL_CLIENT_CERT`
- `CONCOURSE_TERRAFORM_CONSUL_CLIENT_KEY`

e.g. the above working directory might provide the following task params:

```yaml
params:
  CONCOURSE_TERRAFORM_CONSUL_CACERT: consul-certificates/ca/ca-chain.pem
  CONCOURSE_TERRAFORM_CONSUL_CLIENT_CERT: consul-certificates/client.pem
  CONCOURSE_TERRAFORM_CONSUL_CLIENT_KEY: consul-certificates/client-key.pem
```

which when ran with a concourse working directory of `/tmp/build/e55deab7/` will set the below environment values:

```
CONSUL_CACERT=/tmp/build/e55deab7/consul-certificates/ca/ca-chain.pem
CONSUL_CLIENT_CERT=/tmp/build/e55deab7/consul-certificates/client.pem
CONSUL_CLIENT_KEY=/tmp/build/e55deab7/consul-certificates/client-key.pem
```

**note**: these environment variables are not used during the agent join, so you must still also specify any needed certificate paths in `CONSUL_LOCAL_CONFIG`

# tasks

## `plan.yaml`: plan with no output

### inputs

- `concourse-terraform`: _required_. the concourse terraform directory.

- `terraform-source-dir`: _required_. the terraform source directory.

- `state-input-dir`: _optional_. when using local state, the directory containing the state file. you must also configure `STATE_FILE_PATH`. see [managing local state files](#managing-local-state-files)

### outputs

- none

### params

- `TF_WORKING_DIR`: _optional_. path to the terraform working directory. see [providing terraform source files](#providing-terraform-source-files). default: `terraform-source-dir`

- `TF_DIR_PATH`: _optional_. path to the terraform files inside the working directory. see [providing terraform source files](#providing-terraform-source-files). default: `.`

- `STATE_FILE_PATH`: _optional_. when using local state, the path to the input state file. can be relative to the concourse working directory. see [managing local state files](#managing-local-state-files). default: none

- `ERROR_ON_NO_CHANGES`: _optional_. raises an error if applying the plan would result in no changes. set to `false` to disable. default: `true`

- `DESTROY`: _optional_. executes a `-destroy` plan. set to `true` to enable. default: `false`

- `DEBUG`: _optional_. prints command line arguments and increases log verbosity. set to `true` to enable. **may result in leaked credentials**. default: `false`

- `TF_BACKEND_TYPE`: _optional_. generate a terraform `backend.tf` file for this backend type. see [configuring the backend](#configuring-the-backend)

- `TF_BACKEND_CONFIG_{key}`: _optional_. sets `-backend-config` value for `{key}`. see [configuring the backend](#configuring-the-backend)

- `TF_VAR_{key}`: _optional_. terraform input variables in the format described in [providing input variable values](#providing-input-variable-values)

## `apply.yaml`: apply with no plan

**caution**:

- this task may result in orphaned resources if you do not properly manage the state files. to be safe, prefer using a remote backend when possible

- if using a local backend, see [managing local state files](#managing-local-state-files)

- **do not use this task if you want to approve changes before they happen, this will `-auto-approve` any changes, including destructive ones!**

### inputs

- `concourse-terraform`: _required_. the concourse terraform directory.

- `terraform-source-dir`: _required_. the terraform source directory.

- `state-input-dir`: _optional_. when using local state, the directory containing the state file. you must also configure `STATE_FILE_PATH`. see [managing local state files](#managing-local-state-files)

### outputs

- `state-output-dir`: if local state files are present after the apply, they will be placed here as `terraform.tfstate` and `terraform.tfstate.backup`

### params

- `TF_WORKING_DIR`: _optional_. path to the terraform working directory. see [providing terraform source files](#providing-terraform-source-files). default: `terraform-source-dir`

- `TF_DIR_PATH`: _optional_. path to the terraform files inside the working directory. see [providing terraform source files](#providing-terraform-source-files). default: `.`

- `STATE_FILE_PATH`: _optional_. when using local state, the path to the input state file. can be relative to the concourse working directory. see [managing local state files](#managing-local-state-files). default: none

- `DEBUG`: _optional_. prints command line arguments and increases log verbosity. set to `true` to enable. **may result in leaked credentials**. default: `false`

- `TF_BACKEND_TYPE`: _optional_. generate a terraform `backend.tf` file for this backend type. see [configuring the backend](#configuring-the-backend)

- `TF_BACKEND_CONFIG_{key}`: _optional_. sets `-backend-config` value for `{key}`. see [configuring the backend](#configuring-the-backend)

- `TF_VAR_{key}`: _optional_. terraform input variables in the format described in [providing input variable values](#providing-input-variable-values)

## `create-plan.yaml`: create a plan

### inputs

- `concourse-terraform`: _required_. the concourse terraform directory.

- `terraform-source-dir`: _required_. the terraform source directory.

- `state-input-dir`: _optional_. when using local state, the directory containing the state file. you must also configure `STATE_FILE_PATH`. see [managing local state files](#managing-local-state-files)

### outputs

- `plan-output-archive`: an artifact containing the terraform working directory will be placed here

### params

- `TF_WORKING_DIR`: _optional_. path to the terraform working directory. see [providing terraform source files](#providing-terraform-source-files). default: `terraform-source-dir`

- `TF_DIR_PATH`: _optional_. path to the terraform files inside the working directory. see [providing terraform source files](#providing-terraform-source-files). default: `.`

- `STATE_FILE_PATH`: _optional_. when using local state, the path to the input state file. can be relative to the concourse working directory. see [managing local state files](#managing-local-state-files). default: none

- `PLAN_FILE_PATH`: _optional_. path to the terraform plan file inside the working directory. default: `.tfplan`

- `SOURCE_REF`: _optional_. a source ref (e.g. a git commit sha or short sha) to be appended to the output artifact filename. cannot be used with `SOURCE_REF_FILE`. default: none

- `SOURCE_REF_FILE`: _optional_. path to file containing a source ref (e.g. a git commit sha or short sha) to be appended to the output artifact filename. cannot be used with `SOURCE_REF`. default: none

- `ERROR_ON_NO_CHANGES`: _optional_. raises an error if applying the plan would result in no changes. set to `false` to disable. default: `true`

- `DESTROY`: _optional_. creates a `-destroy` plan. set to `true` to enable. default: `false`

- `DEBUG`: _optional_. prints command line arguments and increases log verbosity. set to `true` to enable. **may result in leaked credentials**. default: `false`

- `TF_BACKEND_TYPE`: _optional_. generate a terraform `backend.tf` file for this backend type. see [configuring the backend](#configuring-the-backend)

- `TF_BACKEND_CONFIG_{key}`: _optional_. sets `-backend-config` value for `{key}`. see [configuring the backend](#configuring-the-backend)

- `TF_VAR_{key}`: _optional_. terraform input variables in the format described in [providing input variable values](#providing-input-variable-values)

## `show-plan.yaml`: show a plan

### inputs

- `concourse-terraform`: _required_. the concourse terraform directory.

- `plan-output-archive`: _required_. the directory containing a terraform plan archive generated by `create-plan.yaml`.

	- exactly one file matching `*.tar.gz` must be contained in this directory or the task will fail

### outputs

- none

### params

- `PLAN_FILE_PATH`: _optional_. path to the terraform plan file inside the working directory. default: `.tfplan`

- `DEBUG`: _optional_. prints command line arguments and increases log verbosity. set to `true` to enable. **may result in leaked credentials**. default: `false`

## `apply-plan.yaml`: apply a plan

**caution**:

- this task may result in orphaned resources if you do not properly manage the statefile. to be safe, use a remote backend

- if using a local backend, see [managing local state files](#managing-local-state-files)

### inputs

- `concourse-terraform`: _required_. the concourse terraform directory.

- `plan-output-archive`: _required_. the directory containing a terraform plan archive generated by `create-plan.yaml`.

	- exactly one file matching `*.tar.gz` must be contained in this directory or the task will fail

### outputs

- `state-output-dir`: if local state files are present after the apply, they will be placed here as `terraform.tfstate` and `terraform.tfstate.backup`

### params

- `PLAN_FILE_PATH`: _optional_. path to the terraform plan file inside the working directory. default: `.tfplan`

- `DEBUG`: _optional_. prints command line arguments and increases log verbosity. set to `true` to enable. **may result in leaked credentials**. default: `false`

# development

install python 3.7.1 and requirements from `requirements-dev.txt`

`.vscode/settings.json` will enable linters in vscode

`.vscode/launch.json` contains the vscode launch configuration for `ptvsd`

local builds can be tested and built with the `./scripts/build` and `./scripts/test` scripts, which will automatically build and test all versions listed in the `tf-versions` file

# helper scripts

helper scripts are available in the `./scripts` directory, and expect a working directory of the source code root

## build

builds base and test images

`./scripts/build` supports two run modes:

- `./scripts/build`

	- builds an image for each version in `tf-versions`

- `./scripts/build [VERSION]`

	- builds an image for the specified version

note: to install `ptvsd` in the test image, set the environment variable `PTVSD_INSTALL=1` when running `./scripts/build`, e.g.: `PTVSD_INSTALL=1 ./scripts/build [VERSION]`

## test

runs `unittest` against a test image

`./scripts/test` supports two run modes:

- `./scripts/test`

	- runs `unittest discover` against all versions in `tf-versions`

- `./scripts/test [VERSION] [discover|ARGS]`

	- runs `unittest discover` or `unittest ARGS` against `VERSION`
	- supports `ptvsd` (if installed) by setting environment variables:

		- `PTVSD_ENABLE=1` runs `-m ptvsd -host 0.0.0.0 --port 5678` as the entry point
		- `PTVSD_WAIT=1` enables `--wait` causing the process to wait for the debugger to attach

## run

runs arbitrary args against a test image

`./scripts/run` supports one mode:

- `./scripts/run [VERSION] [ARGS]`

	- runs `ARGS` against `VERSION`

## run_command

runs a `concourse-terraform` command against a test image

also exports relevant variables

`./scripts/run_command ` supports one mode:

- `./scripts/run_command [VERSION] [COMMAND]`

	- runs `COMMAND` against `VERSION`
	- supports `ptvsd` (if installed) by setting environment variables:

		- `PTVSD_ENABLE=1` runs `-m ptvsd -host 0.0.0.0 --port 5678` as the entry point
		- `PTVSD_WAIT=1` enables `--wait` causing the process to wait for the debugger to attach

# automated builds

automated builds are handled by [docker hub](https://hub.docker.com)

custom build hooks are used to build and test every version in the `tf-versions` file

# license

see [LICENSE](LICENSE)
