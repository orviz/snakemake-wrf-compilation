# WRF Installation Pipeline with Snakemake & EasyBuild

This repository provides a generic, reproducible **Snakemake** workflow designed exclusively to automate and monitor the installation of the **WRF (Weather Research and Forecasting)** model within High-Performance Computing (HPC) environments.

The workflow has been designed to implement **checkpoint tracking based on predecible output WRF files**. Consequently, if the compilation process gets interrupted (walltime limits, etc.), Snakemake detects which of those WRF output files (`configure.wrf`, `libwrflib.a`, etc.) are already generated on disk and resumes the installation exactly from the phase it left off.

---

## Prerequisites

Before executing this pipeline, ensure that your HPC environment has the following components pre-installed and available in your terminal path:

1. **Environment Modules:** Lmod or traditional Environment Modules.
2. **EasyBuild:** Installed and initialized (the command `eb --version` should return successfully).
3. **Snakemake:** Version 7.0 or higher installed in your active Python or Conda environment.

---

## Repository Structure

This repository strictly complies with the official **Snakemake Workflow Standard**:

```text
wrf-installation-pipeline/
├── .gitignore               # Excludes logs and temporary Snakemake directories
├── README.md                # This usage guide
├── config/
│   └── config.yaml          # System environment, toolchain, and software path variables
└── workflow/
    └── Snakefile            # DAG workflow with granular physical WRF checkpoints
```

---

## Quick Start

### Environment Setup

This repository uses **Pixi** to guarantee user-space isolated environments without requiring root privileges or pre-installed cluster software.

#### 1. Install Pixi (*One-time setup*)

If Pixi is not available on your (local or HPC login) node, install it locally:

```bash
curl -fsSL https://pixi.sh/install.sh | sh
source ~/.bashrc
```

#### 2. Clone this repository

Clone this project into your scratch or home space inside the cluster:

```bash
git clone https://github.com/orviz/snakemake-wrf-compilation
cd snakemake-wrf-compilation
```

#### 3. Project Initialization

Pixi will automatically read `pixi.toml`, download the exact versions of Snakemake and EasyBuild locked in `pixi.lock`, and create an isolated local environment:

```bash
pixi install
```
### WRF installation

#### 1. Configure system parameters

Open the `config/config.yaml` file and adjust the variables to match your cluster architecture and desired WRF target version:

```yaml
# config/config.yaml
wrf_version: "4.4.1"
toolchain: "foss-2022b"
parallel_type: "dmpar"
eb_software_root: "/home/user/.local/easybuild/software" # Your EasyBuild software root path
```

#### 2. Run the installation pipeline

Use the pre-configured Pixi tasks to run the workflow through Slurm:

* **To run locally (useful for HPC interactive nodes, debugging):**
```bash
pixi run run-local --cores 8
```

* **To run a dry-run validation:**
```bash
pixi run dry-run
```

* **To launch the production build on the Slurm queues:**
```bash
pixi run run-slurm
```

---

## Compilation Phases

The `Snakefile` breaks down the WRF build process into 4 distinct logical blocks:

* **Phase 1 (Fetch):** Downloads and extracts the WRF source code, verifying the presence of the base `./configure` script.
* **Phase 2 (Configure):** Generates target-specific architecture directives, verifying the existence of `configure.wrf`.
* **Phase 3 (Build Core):** Compiles the underlying model framework, verifying the existence of the critical static library file `libwrflib.a`.
* **Phase 4 (Finalize):** Generates the final simulation binaries (`wrf.exe`, `real.exe`, `ndown.exe`, `tc.exe`), performs EasyBuild automated sanity checks, and finalizes the environment module generation.

---

## Integration with Future Simulation Workflows

This repository is built to remain completely decoupled from data production. When building your downstream simulation and data analysis repositories, you can natively invoke this installation pipeline using Snakemake's built-in remote `module` directive:

```python
module install_workflow:
    snakefile: "workflow/Snakefile"
    config: config
    git: "https://github.com/orviz/snakemake-wrf-compilation"
    tag: "v1.0.0"

use rule install_wrf_easybuild from install_workflow as remote_install
```
