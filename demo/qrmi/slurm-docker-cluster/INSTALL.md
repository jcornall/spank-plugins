# Installation

This document describes how to set up a local, container-based, Slurm development environment and how to build and install QRMI and the SPANK plugin in a Slurm cluster.

## Table of Contents
- [Installation](#installation)
  - [Table of Contents](#table-of-contents)
  - [Set Up Local Development Environment](#set-up-local-development-environment)
    - [Prerequisites](#prerequisites)
    - [Creating Docker-based Slurm Cluster](#creating-docker-based-slurm-cluster)
  - [Building and Installing QRMI and the SPANK Plugin](#building-and-installing-qrmi-and-the-spank-plugin)
    - [Running Primitive Job Examples in Slurm Cluster](#running-primitive-job-examples-in-slurm-cluster)
    - [Running Serialized Jobs Using the QRMI Task Runner](#running-serialized-jobs-using-the-qrmi-task-runner)
  - [Troubleshooting](#troubleshooting)
    - [Containers c1 and c2 exiting prematurely](#containers-c1-and-c2-exiting-prematurely)

## Set Up Local Development Environment

### Prerequisites

A container manager such as [Podman](https://podman.io/getting-started/installation.html), [Rancher Desktop](https://rancherdesktop.io/), or [Docker](https://docs.docker.com/get-docker/).

### Creating Docker-based Slurm Cluster

1. Create your local workspace

```bash
mkdir -p <YOUR WORKSPACE>
cd <YOUR WORKSPACE>
```

1. Clone the Slurm Docker Cluster GitHub repository

```bash
git clone -b 0.9.0 https://github.com/giovtorres/slurm-docker-cluster.git
cd slurm-docker-cluster
```

> [!NOTE]
> Slurm Docker Cluster v0.9.0 uses SLURM_TAG defined in slurm-docker-cluster/.env to specify the Slurm version. Currently, SLURM_TAG is set to slurm-25-05-3-1. This corresponds to a tag in Slurm's major release 25.05 from May 2025. Using a Slurm release prior to slurm-24-05-5-1 requires rebuilding the SPANK plugin with -DPRIOR_TO_V24_05_5_1 due to interface changes in slurm-24-05-5-1.

1. Clone the qiskit-community/spank-plugins and qiskit-community/qrmi GitHub repositories

```bash
mkdir shared
pushd shared
git clone https://github.com/qiskit-community/spank-plugins.git
git clone https://github.com/qiskit-community/qrmi.git
popd
```

1. Apply a patch to `slurm-docker-cluster`

```bash
patch -p1 < ./shared/spank-plugins/demo/qrmi/slurm-docker-cluster/file.patch
```

> [!NOTE]
> Rocky Linux 9 is used as default. If you want another operating system, you must apply an additional patch (see below for CentOS 9 and CentOS 10 examples). The patch is used to avoid the Slurm Docker Cluster requirement to include its copyright notice in repositories that copy the Slurm Docker Cluster code.

For CentOS Stream 9:

```bash
patch -p1 < ./shared/spank-plugins/demo/qrmi/slurm-docker-cluster/centos9.patch
```

For CentOS Stream 10:

```bash
patch -p1 < ./shared/spank-plugins/demo/qrmi/slurm-docker-cluster/centos10.patch
```

1. Build the containers

```bash
docker compose build --no-cache
```

> [!NOTE]
> Podman users must install `docker-compose`. MacOS users can do this with `brew install docker-compose`. Ubuntu users may need to use the `docker-compose` command.

1. Spin-up the container cluster

```bash
docker compose up -d
```

Use `docker ps` to check that the following 6 containers are running:

- c2 (Compute Node #2)
- c1 (Compute Node #1)
- slurmctld (Central Management Node)
- slurmdbd (Slurm DB Node)
- login (Login Node)
- mysql (Database node)

> [!IMPORTANT]
> In the event that `docker ps` shows that containers c1 and/or c2 have exited, you may need to inspect the logs to determine the cause of the failure. A list of common issues and fixes is available in the [Troubleshooting](#troubleshooting) section of this guide.

You now have a Slurm cluster as shown below:

<p align="center">
  <img src="../../../docs/images/slurm-docker-cluster.png" width="640">
</p>

## Building and Installing QRMI and the SPANK Plugin

The following steps assume you are building code on **c1** (Compute Node #1). Other nodes are also acceptable.

1. Log in to **c1**:

```bash
docker exec -it c1 bash
```

1. Create a Python virtual environment under shared volume **on c1**:

```bash
python3.12 -m venv /shared/pyenv
source /shared/pyenv/bin/activate
pip install --upgrade pip
```

1. Build and install [QRMI](https://github.com/qiskit-community/qrmi/blob/main/INSTALL.md) **on c1**:

```bash
source ~/.cargo/env
cd /shared/qrmi
pip install -r requirements-dev.txt
maturin build --release
pip install /shared/qrmi/target/wheels/qrmi-*.whl
```

1. Build the [SPANK plugin](../../../plugins/spank_qrmi/README.md) **on c1**:

```bash
cd /shared/spank-plugins/plugins/spank_qrmi
mkdir build
cd build
cmake ..
make
```

This will install QRMI from the [QRMI GitHub repository](https://github.com/qiskit-community/qrmi). If you are building locally for development it might be easier to build QRMI from source mounted at `/shared/qrmi` as shown below:

```bash
cd /shared/spank-plugins/plugins/spank_qrmi
mkdir build
cd build
cmake -DQRMI_ROOT=/shared/qrmi ..
make
```

For pasqal-local resources, make sure to build the SPANK plugin with MUNGE support:

```bash
[root@c1 /]# cd /shared/spank-plugins/plugins/spank_qrmi
[root@c1 /]# mkdir build
[root@c1 /]# cd build
[root@c1 /]# cmake -DENABLE_MUNGE=ON ..
[root@c1 /]# make
```

1. Create the `qrmi_config.json` file:

Modify [this example](https://github.com/qiskit-community/spank-plugins/blob/main/plugins/spank_qrmi/qrmi_config.json.example) to fit your environment and add it to `/etc/slurm` or another location accessible to the Slurm daemons on each compute node you intend to use.

IBM Quantum Platform (IQP) provides limited, free access to IBM Quantum systems. After registering with IBM Cloud and IQP, the list of accessible IBM Quantum systems can be found [here](https://quantum.cloud.ibm.com/computers).

The `qrmi_config.json` file will require an **API key** and a **CRN** for each IQP system. API key instructions can be found [here](https://cloud.ibm.com/iam/apikeys). The CRN for each IQP system can be found [here](https://quantum.cloud.ibm.com/computers). For example, click on "ibm_torino" then open the “Instance access” section for the "ibm_torino" CRN.

> [!WARNING]
> IBM Cloud, and by extension IQP, requires billing information to access the IBM Quantum systems. Contact an administrator to be added to your organisation's IBM Cloud account, which will bypass this requirement.

1. Install the SPANK plugin:

Create `/etc/slurm/plugstack.conf` and ensure it has the following line (assuming `qrmi_config.json` was added to `/etc/slurm`):

```bash
required /shared/spank-plugins/plugins/spank_qrmi/build/spank_qrmi.so /etc/slurm/qrmi_config.json
```

`plugstack.conf`, `qrmi_config.json`, and `spank_qrmi.so` must be installed on the machines that execute slurmd (compute nodes) as well as on the machines that execute job allocation utilities such as salloc, sbatch, etc (login nodes). Refer to the [SPANK documentation](https://slurm.schedmd.com/spank.html#SECTION_CONFIGURATION) for more details.

1. Check the  SPANK plugin installation:

After completing the steps above, `sbatch --help` should show the QPU resource option as shown below:

```bash
[root@c1 /]# sbatch --help

Options provided by plugins:
      --qpu=names             Comma separated list of QPU resources to use.
```

### Running Primitive Job Examples in Slurm Cluster

1. Log in to the **login node**:

```bash
docker exec -it login bash
cd /data # Or another directory shared between the login and compute nodes
```

1. Add your desired QPU (defined in `qrmi_config.json`) to the example job scripts:

```bash
vim /shared/spank-plugins/demo/qrmi/jobs/run_sampler.sh
vim /shared/spank-plugins/demo/qrmi/jobs/run_estimator.sh

#!/bin/bash

...
#SBATCH --qpu=<YOUR CHOSEN QPU>
```

1. For testing purposes, modify the example Python scripts to minimise QPU usage time:

```bash
vim /shared/qrmi/examples/qiskit_primitives/ibm/sampler.py
```

```python
...
circuit = efficient_su2(10, entanglement="linear") # Defaults to 127, reduce to 10 qubits
...
pm = generate_preset_pass_manager(
    optimization_level=1, # Ensure optimization_level is set to 1
    target=target,
)
...
options = {
    "default_shots": 1, # Defaults to 1000, reduce to 1
}
...
```

2. Run the Sampler job on the **login node**:

```bash
sbatch /shared/spank-plugins/demo/qrmi/jobs/run_sampler.sh
```

1. Run the Estimator job on the **login node**:

```bash
sbatch /shared/spank-plugins/demo/qrmi/jobs/run_estimator.sh
```

1. Run the Pasqal job on the **login node**:

```bash
sbatch /shared/spank-plugins/demo/qrmi/jobs/run_pulser_qrmi.sh
```

1. Check the primitive results:

You should find `slurm-{job_id}.out` files in the current directory. For example:

```bash
cat slurm-81.out # Assuming job_id is 81
{'backend_name': 'test_eagle'}
>>> Observable: ['IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII...',
 'IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII...',
 'IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII...',
 'IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII...',
 'IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII...',
 'IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII...',
 'IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII...',
 'IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII...',
 'IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII...',
 'IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII...',
 'IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII...',
 'IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII...',
 'IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII...',
 'IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII...',
 'IIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIIII...', ...]
>>> Circuit ops (ISA): OrderedDict([('rz', 2724), ('sx', 1185), ('ecr', 576), ('x', 288)])
>>> Job ID: 0b1965a6-7473-4efc-aea2-6e2f1c843e5b
>>> Job Status: JobStatus.RUNNING
>>> PrimitiveResult([PubResult(data=DataBin(evs=np.ndarray(<shape=(), dtype=float64>), stds=np.ndarray(<shape=(), dtype=float64>), ensemble_standard_error=np.ndarray(<shape=(), dtype=float64>)), metadata={'shots': 4096, 'target_precision': 0.015625, 'circuit_metadata': {}, 'resilience': {}, 'num_randomizations': 32})], metadata={'dynamical_decoupling': {'enable': False, 'sequence_type': 'XX', 'extra_slack_distribution': 'middle', 'scheduling_method': 'alap'}, 'twirling': {'enable_gates': False, 'enable_measure': True, 'num_randomizations': 'auto', 'shots_per_randomization': 'auto', 'interleave_randomizations': True, 'strategy': 'active-accum'}, 'resilience': {'measure_mitigation': True, 'zne_mitigation': False, 'pec_mitigation': False}, 'version': 2})
  > Expectation value: 0.16554467382152394
  > Metadata: {'shots': 4096, 'target_precision': 0.015625, 'circuit_metadata': {}, 'resilience': {}, 'num_randomizations': 32}
```

You can also verify the successful running of a job via the specific vendor's cloud portal, e.g. quantum.cloud.ibm.com -> Workloads.

### Running Serialized Jobs Using the QRMI Task Runner

It is possible to run JSON-serialized jobs directly using a command line utility called `qrmi_task_runner`. See the [`task_runner` README](https://github.com/qiskit-community/qrmi/tree/main/python/qrmi/tools/task_runner) for more details.

## Troubleshooting

### Containers c1 and c2 exiting prematurely

After running `docker compose up -d`, `docker ps` shows that containers c1 and c2 (the compute nodes) have "Exited" - all other containers show as "Up". You can inspect the logs of an exited container using the following command:

```bash
docker logs <CONTAINER_NAME>
```

If your logs contain the following error message, it is likely that the spank_qrmi.so file has failed to build during the container building stage:

```bash
[2026-06-30T15:17:55.102] error: plugin_load_from_file: dlopen(/shared/spank-plugins/plugins/spank_qrmi/build/spank_qrmi.so): /shared/spank-plugins/plugins/spank_qrmi/build/spank_qrmi.so: cannot open shared object file: No such file or directory
```

This issue can be fixed by building the missing plugin file. However, the file will need to be compatible with the same version of glibc in the containers.

1. Start a disposable container terminal (this will use the same image and mount the same shared directory as the cluster):

```bash
docker compose run --rm --entrypoint bash c1
```

1. Inside the container, build the missing plugin file:

```bash
cd /shared/spank-plugins/plugins/spank_qrmi
rm -rf build
mkdir build
cd build
cmake ..
make
```

1. Exit the container. The generated file will be stored in the local `/shared` directory:

```bash
exit
```

1. Restart the container cluster:

```bash
docker compose down
docker compose up -d
```

Execute `docker ps`. If successful, containers c1 and c2 will now appear as "Up". You can continue on to [the next steps](#building-and-installing-qrmi-and-the-spank-plugin).
