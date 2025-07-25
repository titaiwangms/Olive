# How To Configure Systems

A system is a environment concept (OS, hardware spec, device platform, supported EP) that a Pass is run in or a Model is evaluated on.

There are five systems in Olive: **local system**, **Python environment system**, **Docker system**, **AzureML system**, **Isolated ORT system**. Each system is categorized in one of two types of systems: **host** and **target**. A **host** is the environment where the Pass is run, and a **target** is the environment where the Model is evaluated. Most of time, the **host** and **target** are the same, but they can be different in some cases. For example, you can run a Pass on a local machine with a CPU and evaluate a Model on a remote machine with a GPU.

## Accelerator Configuration

For each **host** or **target**, it will represent list of accelerators that are supported by the system. Each accelerator is represented by a dictionary with the following attributes:

- `device`: The device type of the accelerator. It could be "cpu", "gpu", "npu", etc.
- `execution_providers`: The execution provider list that are supported by the accelerator. For example, `["CUDAExecutionProvider", "CPUExecutionProvider"]`.

The **host** only use the `device` attribute to run the passes. Instead, the **target** uses both `device` and `execution_providers` attributes to run passes or evaluate models.

## Local System

The local system represents the local machine where the Pass is run or the Model is evaluated and it only contains the accelerators attribute.

```json
{
    "systems": {
        "local_system" : {
            "type": "LocalSystem",
            "accelerators": [
                {
                    "device": "cpu",
                    "execution_providers": ["CPUExecutionProvider"]
                }
            ]
        }
    },
    "engine": {
        "target": "local_system"
    }
}
```

```{Note}
* Local system doesn't support `olive_managed_env`.
* The accelerators attribute for local system is optional. If not provided, Olive will get the available execution providers installed in the current local machine and infer its `device`.
* For each accelerator, either `device` or ``execution_providers` is optional but not both if the accelerators are specified. If `device` or `execution_providers` is not provided, Olive will infer the `device` or `execution_providers` if possible.

Most of time, the local system could be simplified as below:

    {
        "type": "LocalSystem"
    }

In this case, Olive will infer the `device` and `execution_providers` based on the local machine. Please note the `device` attribute is required for **host** since it will not be inferred for host system.
```

## Python Environment System

The python environment system represents the python virtual environment. The python environment system is configured with the following attributes:

- `accelerators`: The list of accelerators that are supported by the system.
- `python_environment_path`: The path to the python virtual environment, which is required for native python system.
- `environment_variables`: The environment variables that are required to run the python environment system. This is optional.
- `prepend_to_path`: The path that will be prepended to the PATH environment variable. This is optional.
- `olive_managed_env`: A boolean flag to indicate if the environment is managed by Olive. This is optional and defaults to False.
- `requirements_file`: The path to the requirements file, which is only required and used when `olive_managed_env = True`.

### Native Python Environment System


Here are the examples of configuring the general Python Environment System.

```json
{
    "systems"  : {
        "python_system" : {
            "type": "PythonEnvironment",
            "python_environment_path": "/home/user/.virtualenvs/myenv/bin",
            "accelerators": [
                {
                    "device": "cpu",
                    "execution_providers": [
                        "CPUExecutionProvider",
                        "OpenVINOExecutionProvider"
                    ]
                }
            ]
        }
    },
    "engine": {
        "target": "python_system"
    }
}
```

```{Note}
- The python environment must have `olive-ai` installed if `olive_managed_env = False`.
- The accelerators for python system is optional. If not provided, Olive will get the available execution providers installed in current python virtual environment and infer its `device`.
- For each accelerator, either `device` or `execution_providers` is optional but not both if the accelerators are specified. If `device` or `execution_providers` is not provided, Olive will infer the `device` or `execution_providers` if possible.
```

### Managed Python Environment System

When `olive_managed_env = True`, Olive will manage the python environment by installing the required packages from the `requirements_file`. As the result, the `requirements_file` is required and must be provided.

For managed python environment system, Olive can only infer the onnxruntime from the following onnxruntime execution providers:

- CPUExecutionProvider: (*onnxruntime*)
- CUDAExecutionProvider: (*onnxruntime-gpu*)
- TensorrtExecutionProvider: (*onnxruntime-gpu*)
- OpenVINOExecutionProvider: (*onnxruntime-openvino*)
- DmlExecutionProvider: (*onnxruntime-directml*)

```json
{
    "type": "PythonEnvironment",
    "accelerators": [
        {
            "device": "cpu",
            "execution_providers": [
                "CPUExecutionProvider",
                "OpenVINOExecutionProvider"
            ]
        }
    ],
    "olive_managed_env": true,
}
```

## Docker System

The docker system represents the docker container where the Pass is run or the Model is evaluated. It can be configured as a native system or a managed system. The docker system is configured with the following attributes:

* `accelerators`: The list of accelerators that are supported by the system.
* `local_docker_config`: The configuration for the local docker system, which includes the following attributes:

    * `image_name`: The name of the docker image.
    * `build_context_path`: The path to the build context.
    * `dockerfile`: The path to the Dockerfile.

* `requirements_file`: The path to the requirements file. If provided, Olive will install the required packages from the requirements file in the docker container.
* `olive_managed_env`: A boolean flag to indicate if the environment is managed by Olive. This is optional and defaults to False.

```{Note}
* the `build_context_path`, `dockerfile` and `requirements_file` cannot be ``None`` at the same time.
* The docker container must have `olive-ai` installed.
* The ``device`` and ``execution_providers`` for docker system is mandatory. Otherwise, Olive will raise an error.
```

### Prerequisites


1. Docker Engine installed on the host machine.
2. docker extra dependencies installed. `pip install olive-ai[docker]` or `pip install docker`


### Native Docker System

```json
{
    "type": "Docker",
    "local_docker_config": {
        "image_name": "olive",
        "build_context_path": "docker",
        "dockerfile": "Dockerfile"
    },
    "accelerators": [
        {
            "device": "cpu",
            "execution_providers": ["CPUExecutionProvider"]
        }
    ]
}
```

### Managed Docker System

When `olive_managed_env = True`, Olive will manage the docker environment by installing the required packages from the `requirements_file` in the docker container if provided.
From the time being, Olive only supports the following base Dockerfiles based on input execution providers:

* CPUExecutionProvider: (*Dockerfile.cpu*)
* CUDAExecutionProvider: (*Dockerfile.gpu*)
* TensorrtExecutionProvider: (*Dockerfile.gpu*)
* OpenVINOExecutionProvider: (*Dockerfile.openvino*)

A typical managed Docker system can be configured by the following example:

```json
{
    "type": "Docker",
    "accelerators": [
        {
            "device": "cpu",
            "execution_providers": [
                "CPUExecutionProvider",
                "OpenVINOExecutionProvider"
            ]
        }
    ],
    "olive_managed_env": true,
    "requirements_file": "mnist_requirements.txt"
}
```

## AzureML System

> ⚠️ **DEPRECATION WARNING**: This feature will be deprecated in the next release.

The AzureML system represents the Azure Machine Learning workspace where the Pass is run or the Model is evaluated. It can be configured as a native system or a managed system. The AzureML system is configured with the following attributes:

* `accelerators`: The list of accelerators that are supported by the system, which is required.
* `aml_compute`: The name of the AzureML compute, which is required.
* `azureml_client_config`: The configuration for the AzureML client, which includes the following attributes:

    * `subscription_id`: The subscription id of the AzureML workspace.
    * `resource_group`: The resource group of the AzureML workspace.
    * `workspace_name`: The name of the AzureML workspace.

* `aml_docker_config`: The configuration for the AzureML docker system, which includes the following attributes:

    * `base_image`: The base image for the AzureML environment.
    * `dockerfile`: The path to the Dockerfile of the AzureML environment.
    * `build_context_path`: The path to the build context of the AzureML environment.
    * `conda_file_path`: The path to the conda file.
    * `name`: The name of the AzureML environment.
    * `version`: The version of the AzureML environment.

* `aml_environment_config`: The configuration for the AzureML environment, which includes the following attributes:

    * `name`: The name of the AzureML environment.
    * `version`: The version of the AzureML environment.
    * `label`: The label of the AzureML environment.

* `requirements_file`: The path to the requirements file. If provided, Olive will install the required packages from the requirements file in the AzureML environment.
* `tags`: The tags for the AzureML environment. This is optional.
* `resources`: The resources dictionary for the AzureML environment. This is optional.
* `instance_count`: The instance count for the AzureML environment. Default is 1.
* `datastore`: The datastore name where to export artifacts. Default is `workspaceblobstore`.
* `olive_managed_env`: A boolean flag to indicate if the environment is managed by Olive. This is optional and defaults to False.

```{Note}
* Both `aml_docker_config` and `aml_environment_config` cannot be ``None`` at the same time.
* If `aml_environment_config` is provided, Olive will use the existing AzureML environment with the specified name, version and label.
* Otherwise, Olive will create a new AzureML environment using the `aml_docker_config` configuration.
* The `azureml_client_config` will be propagated from engine `azureml_client` if not provided.
* The `requirements_file` is only used when `olive_managed_env = True` to install the required packages in the AzureML environment.
* The ``device`` and ``execution_providers`` for AzureML system is mandatory. Otherwise, Olive will raise an error.
```

### Prerequisites

1. The azureml extra dependencies are installed with `pip install olive-ai[azureml]` or `pip install azure-ai-ml azure-identity`.
1. AzureML Workspace with necessary compute created. Read [Azure Machine Learning Workspace](https://learn.microsoft.com/en-us/azure/machine-learning/concept-workspace) for more details. Download
the workspace config json.
1. In order to submit and run a job on AzureML Workspace, you need either of the following roles assigned to the workspace: `AzureML Data Scientist`, `Contributor` or `Owner`. Find more details about permission requirement in this [document](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-assign-roles?view=azureml-api-2&tabs=team-lead#default-roles).

### Native AzureML System

```json
{
    "type": "AzureML",
    "accelerators": [
        {
            "device": "gpu",
            "execution_providers": [
                "CUDAExecutionProvider"
            ]
        }
    ],
    "aml_compute": "gpu-cluster",
    "aml_docker_config": {
        "base_image": "mcr.microsoft.com/azureml/openmpi4.1.0-cuda11.8-cudnn8-ubuntu22.04",
        "conda_file_path": "conda.yaml"
    },
    "aml_environment_config": {
        "name": "myenv",
        "version": "1"
    }
}
```

### AzureML Readymade Systems

There are some readymade systems available for AzureML. These systems are pre-configured with the necessary.

```json
{
    "type": "AzureNDV2System",
    "accelerators": [
        {"device": "gpu", "execution_providers": ["CUDAExecutionProvider"]},
        {"device": "cpu", "execution_providers": ["CPUExecutionProvider"]},
    ],
    "aml_compute": "gpu-cluster",
    "aml_docker_config": {
        "base_image": "mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu22.04",
        "conda_file_path": "conda.yaml"
    }
}
```

System alias list:

- AzureND12SSystem:
  - sku: "STANDARD_ND12S"
  - num_cpus: 12
  - num_gpus: 2

- AzureND24RSSystem:
  - sku: "STANDARD_ND24RS"
  - num_cpus: 24
  - num_gpus: 4

- AzureND24SSystem:
  - sku: "STANDARD_ND24S"
  - num_cpus: 24
  - num_gpus: 4

- AzureNDV2System:
  - sku: "STANDARD_ND40RS_V2"
  - num_cpus: 40
  - num_gpus: 8

- AzureND6SSystem:
  - sku: "STANDARD_ND6S"
  - num_cpus: 6
  - num_gpus: 1

- AzureND96A100System:
  - sku: "STANDARD_ND96AMSR_A100_V4"
  - num_cpus: 96
  - num_gpus: 8

- AzureND96ASystem:
  - sku: "STANDARD_ND96ASR_V4"
  - num_cpus: 96
  - num_gpus: 8


```{Note}
The accelerators specified in the readymade systems will be filtered against the devices supported by the readymade system. If the specified device is not supported by the readymade system, Olive will filter out the accelerator.
In above example, the readymade system supports only GPU. Therefore, the final accelerators will be ``[{"device": "gpu", "execution_providers": ["CUDAExecutionProvider"]}]`` and the CPU will be filtered out.
```

### Managed AzureML System

When `olive_managed_env = True`, Olive will manage the AzureML environment by installing the required packages from the `requirements_file` in the AzureML environment if provided.

Currently, Olive supports the following base Dockerfiles based on input execution providers:

* CPUExecutionProvider: (*Dockerfile.cpu*)
* CUDAExecutionProvider: (*Dockerfile.gpu*)
* TensorrtExecutionProvider: (*Dockerfile.gpu*)
* OpenVINOExecutionProvider: (*Dockerfile.openvino*)

A typical managed AzureML system can be configured by the following example:

```json
{
    "systems": {
        "azureml_system": {
            "type": "AzureML",
            "accelerators": [
                {
                    "device": "cpu",
                    "execution_providers": [
                        "CPUExecutionProvider",
                        "OpenVINOExecutionProvider"
                    ]
                }
            ],
            "azureml_client_config": {
                "subscription_id": "subscription_id",
                "resource_group": "resource_group",
                "workspace_name": "workspace_name"
            },
            "aml_compute": "cpu-cluster",
            "requirements_file": "mnist_requirements.txt",
            "olive_managed_env": true,
        }
    },
    "engine": {
        "target": "azureml_system",
    }
}
```

## Isolated ORT System

The isolated ORT system represents the isolated ONNX Runtime environment in which the `olive-ai` is not installed. It can only be configured as a target system. The isolated ORT system is configured with the following attributes:

* `accelerators`: The list of accelerators that are supported by the system.
* `python_environment_path`: The path to the python virtual environment.
* `environment_variables`: The environment variables that are required to run the python environment. This is optional.
* `prepend_to_path`: The path that will be prepended to the PATH environment variable. This is optional.

```json
{
    "type": "IsolatedORT",
    "python_environment_path": "/home/user/.virtualenvs/myenv/bin",
    "accelerators": [{"device": "cpu"}]
}
```

```{Note}
* Isolated ORT System does not support `olive_managed_env` and can only be used to evaluate ONNX models.
* The accelerators for Isolated ORT system is optional. If not provided, Olive will get the available execution providers installed in current virtual environment and infer its device.
* For each accelerator, either `device` or `execution_providers` is optional but not both if the accelerators are specified. If `device` or `execution_providers` is not provided, Olive will infer the ``device`` or `execution_providers` if possible.
```
