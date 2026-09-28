# Automate Kubernetes Deployment with Terraform and Ansible

Terraform provisions an Amazon EKS cluster; Ansible then deploys an Nginx app into it. The two steps are kept in separate repos:

| Step | Tool | Repo |
|---|---|---|
| 1. Provision VPC + EKS cluster | Terraform | [OyedunOye/terraform-project (`feature/eks`)](https://github.com/OyedunOye/terraform-project/tree/feature/eks) |
| 2. Create namespace and deploy the app | Ansible | this repo |

```
terraform-project (feature/eks)          this repo
────────────────────────────────         ─────────────────────────────────
terraform apply
  ├─ VPC  (my-app-vpc)
  └─ EKS  (my-app-eks-cluster)  ──►  kubeconfig file  ──►  deploy-to-k8s.yaml
                                                             ├─ create namespace `my-app`
                                                             └─ apply nginx Deployment + Service
```

Ansible does not create the cluster. It only talks to the cluster's API through a kubeconfig file.

## Tech Stack
Terraform, Ansible, Linux, Kubernetes, AWS EKS.

## Repo contents

| File | Purpose |
|---|---|
| `deploy-to-k8s.yaml` | The playbook |
| `nginx-config.yaml` | Kubernetes manifest applied by the playbook (Deployment + Service) |
| `project-vars` | Paths used by the playbook (virtualenv interpreter, kubeconfig), both relative to the project |
| `.gitignore` | Excludes the generated `kubeconfig` |
| `Pipfile`, `Pipfile.lock` | Python dependencies (`pyyaml`, `kubernetes`, `jsonpatch`) for the `kubernetes.core` modules |

## The cluster being targeted

Summarised from the Terraform repo (see its README for the full picture):

| Item | Value |
|---|---|
| Cluster name | `my-app-eks-cluster` |
| VPC | `my-app-vpc`, public + private subnets across multiple AZs, single NAT gateway |
| Kubernetes version | `1.36` |
| Nodes | Managed node group `dev` (AL2023 x86_64), min 1 / max 3 / desired 2, in the private subnets |
| API endpoint | Public access enabled, so the playbook can reach it from a local machine |
| Access | The identity that ran `terraform apply` is granted cluster admin |
| Add-ons | `coredns`, `kube-proxy`, `vpc-cni`, `eks-pod-identity-agent` |

Because the kubeconfig authenticates through the AWS CLI, run the playbook with the **same AWS identity** that created the cluster (or one that has been given an access entry on it).

## What the playbook does

The play targets `localhost` (Ansible uses a local connection for it implicitly, since all the work is API calls to the cluster) and has two tasks, both using `kubernetes.core.k8s` with `state: present`:

1. **Create a k8s namespace**: creates the `my-app` namespace.
2. **Deploy nginx app**: applies every object in `nginx-config.yaml` into the `my-app` namespace.

Both tasks are idempotent: re-running reports `ok` instead of `changed` when the cluster already matches.

`nginx-config.yaml` defines:

- a `Deployment` named `nginx` with 1 replica running the `nginx` image on port 80
- a `Service` named `nginx` of type `LoadBalancer` on port 80, which makes AWS provision an Elastic Load Balancer with a public DNS name in front of the pods

### Why the Python interpreter is overridden

The `kubernetes.core` modules need `pyyaml`, `kubernetes` and `jsonpatch` on the machine running the play (here `localhost`). They are installed in a `pipenv` virtualenv rather than system-wide, created with `pipenv install`. Pipenv puts it in `~/.local/share/virtualenvs/` under a name that differs per machine, so `project-vars` looks the path up at run time (`pipenv --venv`) and uses it for `ansible_python_interpreter`. The playbook loads it with `vars_files`, so nothing needs editing on a new machine.

Without it, the module fails with an error about the `kubernetes` Python library being missing.

This is set as a variable rather than through Ansible's `interpreter_python` config setting on purpose: Ansible gives the implicit `localhost` its own `ansible_python_interpreter`, which takes precedence over that setting, whereas a play-level variable overrides it.

## Paths

Nothing in the playbook needs editing for a new machine. The paths are set in `project-vars`:

| Variable | Resolves to | Created by |
|---|---|---|
| `ansible_python_interpreter` | `<pipenv virtualenv>/bin/python3`, resolved by `pipenv --venv` | step 2 |
| `kubeconfig_path` | `<project>/kubeconfig` | step 3 |

The manifest is resolved as `{{ playbook_dir }}/nginx-config.yaml`. `kubeconfig` is generated per cluster, so it is git-ignored.

## Step-by-step

### 1. Provision the cluster with Terraform

```bash
git clone -b feature/eks https://github.com/OyedunOye/terraform-project.git
cd terraform-project
# create terraform.tfvars first (it is git-ignored) - see that repo's README
terraform init
terraform apply
```

Besides the variables in the Terraform README's example (`vpc_cidr_block`, subnet CIDRs, `aws_availability_zones`, `instance_type`), `variables.tf` also requires `console_account_arn` and `console_account_policy`, which grant an additional IAM principal access to the cluster. Set these in `terraform.tfvars` too.

### 2. Install the Ansible-side dependencies

```bash
pipenv install                                      # installs pyyaml, kubernetes, jsonpatch
ansible-galaxy collection install kubernetes.core   # skip if you use the full `ansible` package
```

### 3. Generate the kubeconfig

```bash
aws eks update-kubeconfig \
  --name my-app-eks-cluster \
  --region <your-region> \
  --kubeconfig ./kubeconfig
```

If you use a different path, update `kubeconfig_path` in `project-vars`. Recreate this file whenever you recreate the cluster.

### 4. Run the playbook

```bash
ansible-playbook deploy-to-k8s.yaml
```

No `-i` is needed because the play targets `localhost`, which Ansible always knows about implicitly.

### 5. Verify the deployment

```bash
export KUBECONFIG=$PWD/kubeconfig
kubectl get pods,svc -n my-app
```

The pod should be `Running`, and the `nginx` service should show an `EXTERNAL-IP` (an AWS ELB hostname; it can take a couple of minutes to appear). Opening that hostname in a browser should show the default Nginx welcome page.

## Cleaning up

Delete the Kubernetes objects **before** destroying the cluster with Terraform:

```bash
kubectl delete namespace my-app        # removes the Service, which in turn removes the AWS load balancer
cd ../terraform-project && terraform destroy
```

The load balancer is created by Kubernetes, not Terraform, so Terraform does not know about it. If it is still there, `terraform destroy` can hang or fail while deleting the VPC because the ELB is still using its subnets and network interfaces.

