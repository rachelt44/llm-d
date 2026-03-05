---
name: run-llm-d-benchmark
description: Runs a workload against an already deployed llm-d stack (high-performance distributed LLM inference), using llm-d-benchmark tooling.
---

# Run llm-d Benchmark Skill

## purpose

Run a specific benchmark harness and workload against an already deployed llm-d stack (high-performance distributed LLM inference). Useful for evaluating the performance of the llm-d stack. When benchmark terminates, store benchmarking results in a local storage. Optioanlly create scripts on top of the raw results to analyze them.

## Workflow

### Step 1: Locate the llm-d stack and set namespace accordingly

Locate the llm-d stack according to the following logic:

1. If a NAMESPACE environment variable is specified, the llm-d stack is assumed to be deployed there
2. If an oc project exists, the stack is assumed to be deployed in the current oc project.
3. If none of the above holds, ask the user for the NAMESPACE where it is deployed.
4. Make sure the NAMESPACE environment variable is set.

Verify that the stack is indeed deployed in the detected or provided namespace using kubectl commands. If you cannot locate the stack, ask the user to deploy one and refer to the llm-d-kubernetes deployment skill.

### Step 2: Verify the existence of a storage class

Make sure Kubernetes storage class for PVCs exists in the namespace, using the command `kubectl get pvc -n $NAMESPACE`. If it doesn't exist, create a PVC after asking the user to provide its required name. make sure the BENCHMARK_PVC environment variable is set to the name of the PVC.

### Step 3: Determine the template configuration file for benchmarking and instantiate it

Display to the user the available benchmarking template yaml files located here:

https://github.com/llm-d/llm-d/tree/main/guides/benchmark

Let the user select one of the displayed template configuration files.

Instantiate it using the command `envsubst < inference_scheduling_guide_template.yaml > config.yaml`

### Step 4: Select benchmark harness 

Ask the user to either confirm the harness appearing in the selected configuration file, or select a new one. The available harnesses are:
- `inference-perf` 
- `guidellm`
- `inferencemax`
- `vllm-benchmark`

If a non-default harness is used, update the instantiated configuration file with the selected harness.

### Step 5: Select or generate workload to use

Ask the user to select one of the following options:

1. Confirm the existing workload specified in the config file (default)
2. Provide a link or a path to another workload
3. Provide a path to a trace to replay
4. provide a workload description. In this case, you should automatically generate a workload based on it.

If a non-default workload is used, update the instantiated configuration file with the selected or generated workload.

### Step 6: Verify the model to use

Display the model name specified in the configuration file to the user for verification.

If the user wants to change the model, check which models are available in the cluster. Let the user select one of them, and update the instantiated configuration file with the selected model name.

### Step 7: Verify the final configuration

Display the instantiated configuration file to the user for verification. If the user wants to change any of the configuration parameters, update the instantiated configuration file based on user feedback.

**Common customizations include**:

 **Model name**: Which model to benchmark (default: Qwen/Qwen3-32B)
- **Workload type**: Type of synthetic data (random, shared_prefix, etc.)
- **Load stages**: Request rates and durations
- **Input/Output distribution**: Prompt and response length parameters
  - `input_distribution`: min, max, mean, std, total_count
  - `output_distribution`: min, max, mean, std, total_count
- **API settings**: Streaming mode, completion type
- **Parallelism**: Number of parallel workload launcher pods

### Step 8: Obtain the benchmarking script 

Use the following commands to download the benchmarking script:

`curl -L -O https://raw.githubusercontent.com/llm-d/llm-d-benchmark/main/existing_stack/run_only.sh`
`chmod u+x run_only.sh`

### Step 9: Run benchmarking

Use the command `./run_only.sh -c config.yaml`, monitor its progress and wait for its completion.

Note that the benchmark harness pod may still be running also after the benchmarking run is completed.

### Step 10: Locate and save results

Ask the user for a path to store the results. Save the benchmarking results by copying them from the BENCHMARK_PVC to a local `results` directory inside the path specified by the user.  This step requires locating the results of the current benchmarkr run in the BENCHMARK_PVC. This can be performed using the command ` kubectl exec -n $NAMESPACE llmdbench-harness-launcher -- ls -ltr /requests/`.

### Step 11: Run analyses and save them

Ask the user whether any analysis of raw results is requested. For example, create specific graphs or tables from raw metric reporting. if the user provides an analysis description, create corresponding analysis scripts, run them, and store them along with their results in the `analysis` directory inside the `results` directory.


#### Execution Rules

1. **Run every command** from Steps 1–11 in order using `execute_command`.
2. **After each command**, inspect the output:
   - If the command **succeeds** → proceed to the next command.
   - If the command **fails** → diagnose the error, apply a fix, and re-run before continuing.
3. **Never skip a step** — each step depends on the previous one succeeding.

**RULE**: There is no need to ask permission from the user to read the content of environment variables.

> ## 🔔 ALWAYS NOTIFY THE USER BEFORE CREATING ANYTHING
>
> **RULE**: Before creating ANY resource — including PVCs, files, or any Kubernetes object — you MUST first tell the user what you are about to create and why.

 **RULE**: Before deleting or overriding ANY resources — including PVCs, files, or any Kubernetes object — you MUST first get confirmation from the user.

> **Format to use before every creation action**:
> > "I am about to create `<resource-type>` named `<name>` because `<reason>`. Proceeding now."
>
> **Examples**:
> - "I am about to create PVC `llm-d-kv-cache-storage` (18000Gi, storage class: default) because no PVC was found in namespace `llm-d`. Proceeding now."
> - "I am about to create file `${LLMD_PATH}/pvc.yaml` with the PVC definition. Proceeding now."
>
> **Never silently create resources.** If you are unsure whether a resource already exists, check first, then notify before acting.

## What Not To Do

Critical rules to follow when configuring and running benchmarking on llm-d:

1. **Do NOT change cluster-level definitions** — All changes must be made exclusively inside the designated project namespace. Never modify cluster-wide resources (e.g., ClusterRoles, ClusterRoleBindings, StorageClasses, Nodes, or any resource outside the target namespace). Scope every `kubectl apply`, `helm install`, and `helmfile apply` command to the target namespace using `-n ${NAMESPACE}`.

2. **Do NOT modify any existing code you did not create** — Only create new files and modify them as needed. Never edit pre-existing files in the repository (e.g., existing `values.yaml`, `helmfile.yaml`, `httproute.yaml`, `README.md`, or any other committed file). If customization is required, create a new file (e.g., `values-custom.yaml`, `httproute-custom.yaml`) and reference it instead.


## When to Use This Skill

Activate this skill when users need to:
- Run benchmarking against a deployed llm-d stack on Kubernetes
- Analyze llm-d specific-configuration serving performance 
- Optimize llm-d serving performance
- Choose between different configuration strategies

## Prerequisites

### Client Setup
**Guide**: `https://raw.githubusercontent.com/llm-d/llm-d/guides/prereq/client-setup/README.md`

Required tools:
- kubectl
- helm
- helmfile
- git


## Security Considerations

- Run benchmarking in isolated namespaces
- Use RBAC for access control
- Secure HuggingFace tokens as Kubernetes secrets
- Enable network policies for pod-to-pod communication
- Use TLS for gateway ingress
- Audit all deployments and changes

