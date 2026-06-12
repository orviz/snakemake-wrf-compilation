# HPC Cluster Configuration Guide (Slurm Profile)

This directory contains the infrastructure configuration files required to map the generic WRF installation pipeline to your local High-Performance Computing (HPC) cluster resources.

To ensure portability, this workflow decouples **abstract job requirements**, defined inside `workflow/Snakefile`, from **HPC-specific cluster flags**, defined here in the Slurm profile.

---

## Slurm Parameter Discovery (step-by-step)

Before launching the pipeline on a new cluster, you must discover your local partition names, account billings, and permissions. Run the following commands on your cluster's **login node** to extract the values needed for `config/profiles/slurm_cluster/config.yaml`. 

Note that, depending on the HPC site policies, some Slurm commands might not be available. In that case you will need to find out alternative ways to obtain the required info.

### 1. Identify Your Authorized Partition (`slurm_partition`)
To discover which queues/partitions you are allowed to submit jobs to, execute:
```bash
squeue -u \$USER -o "%P" | grep -v PARTITION | sort -u
```
*If global cluster queries are locked down by your sysadmins, you can run a 1-minute interactive tracking probe instead:*
```bash
srun --time=00:01:00 --pty bash -c "squeue -j \$SLURM_JOB_ID -o %P"
```
* **Action**: Take the resulting partition string (e.g., `compute`, `batch`, `normal`) and update the `slurm_partition` fields under both `default-resources` and `set-resources` blocks.

### 2. Identify Your Project Billing Account (`slurm_account`)
Most institutional clusters require a project allocation string to track CPU-hour consumption. To see your active budget links, run:
```bash
sacctmgr -p show associations user=\$USER format=Account | grep -v "Account"
```
* **Action**: Copy your target project group name (e.g., `computacion`, `physics_dept`) into the `slurm_account` field. 
* **Note**: If your cluster configuration does not require accounting flags, you can completely delete the `slurm_account` line from the `.yaml` profile.

---

## Profile Customization File (`profiles/slurm_cluster/config.yaml`)

Open `profiles/slurm_cluster/config.yaml` and update it using your discovered cluster credentials. Keep the overall configuration structure intact to preserve Snakemake executor integrations:

```yaml
# Target execution framework
executor: slurm

# Global defaults applied to lightweight rules (Phases 1 & 2)
default-resources:
  slurm_partition: "YOUR_DISCOVERED_PARTITION"
  slurm_account: "YOUR_BILLING_ACCOUNT"
  runtime: 60  # Default safe walltime in minutes (1 hour)

# Cluster overrides for the heavy-duty WRF compilation rule (Phase 3 & 4)
set-resources:
  wrf_core_libraries_checkpoint:
    slurm_partition: "YOUR_DISCOVERED_PARTITION"

# Snakemake orchestrator pacing parameters
restart-times: 1                # Auto-retry failed cluster jobs once (e.g., due to walltime timeout)
max-jobs-per-second: 2          # Prevents throttling the Slurm controller API
max-status-checks-per-second: 10
local-cores: 1                  # Keeps file tracking tasks on a single login node core
```

---

## Validation Checklist

After editing the configuration values according to your cluster specifications, **always perform a first non-invasive validation run** from the repository's root path:

```bash
pixi run dry-run
```

Ensure the dry-run output correctly parses your partition names and maps.