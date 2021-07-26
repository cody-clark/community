---
title: Upgrade Anthos Service Mesh on GKE with Terraform
description: Use Terraform to deploy a GKE cluster and Anthos Service Mesh with in-cluster control plane and perform a revision-based upgrade.
author: cloud-pharaoh
tags: Kubernetes Engine, ASM
date_published: 2021-06-23
---

Amina Mansour | Solutions Architect | Google

<p style="background-color:#CAFACA;"><i>Contributed by Google employees.</i></p>

This tutorial shows you how to install Anthos Service Mesh 1.9 with an in-cluster control plane on a GKE cluster using the [GKE Anthos Service Mesh Terraform submodule](https://github.com/terraform-google-modules/terraform-google-kubernetes-engine/tree/master/modules/asm) then upgrade to 1.10 following the revision upgrade process (or "canary" upgrades in Istio).

With a revision-based upgrade, you install a new revision of the control plane alongside the existing control plane. When installing the new version, the script includes a revision label that identifies the new control plane.

You then migrate to the new version by setting the same revision label on your workloads and performing a rolling restart to re-inject the proxies so they use
the new Anthos Service Mesh version and configuration. With this approach, you can monitor the effects of the upgrade on a small percentage of your workloads.

After testing your application, you can migrate all traffic to the new version or rollback the changes. This approach is much safer than doing an in-place upgrade where new control plane components replace the previous version.

## Objectives

- Use the GKE Anthos Service Mesh Terraform submodule to
  - Create a Virtual Private Cloud (VPC)
  - Create a GKE cluster
  - Install Anthos Service Mesh 1.9
- Deploy the [Online Boutique](https://cloud.google.com/service-mesh/docs/onlineboutique-install-kpt) sample app on an Anthos Service Mesh labeled `online-boutique` namespace
- Upgrade to Anthos Service Mesh 1.10 using revision upgrade process
- Cleanup or destroy all resources via Terraform

## Costs 

This tutorial uses the following Google Cloud products:

*   [Kubernetes Engine](https://cloud.google.com/kubernetes_engine)
*   [Anthos Service Mesh](https://cloud.google.com/service-mesh)
*   [Cloud Storage](https://cloud.google.com/storage)

Use the [pricing calculator](https://cloud.google.com/products/calculator) to generate a cost estimate based on your projected usage.

## Before you begin

1. [Select or create a Google Cloud project](https://console.cloud.google.com/projectselector2).
1. [Very that billing is enabled](https://cloud.google.com/billing/docs/how-to/modify-project) for your project.
1. Enable the required APIs:

    ```
    gcloud services enable cloudresourcemanager.googleapis.com \
      container.googleapis.com
    ```

### Setting up your environment

1. Create environment variables:

    ```
    export PROJECT_ID=YOUR_PROJECT_ID

    gcloud config set project ${PROJECT_ID}
    export PROJECT_NUM=$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')
    export CLUSTER_1=gke-central
    export CLUSTER_1_ZONE=us-central1-a
    export CLUSTER_1_CTX=gke_${PROJECT_ID}_${CLUSTER_1_ZONE}_${CLUSTER_1}
    export WORKLOAD_POOL=${PROJECT_ID}.svc.id.goog
    export MESH_ID="proj-${PROJECT_NUM}"
    export TERRAFORM_SA="terraform-sa"
    export ASM_MAJOR_VERSION=1.9
    export ASM_MAJOR_VERSION_UPGRADE=1.10
    ```
    
    where you replace `YOUR_PROJECT_ID` with your project ID.

1. Create a WORKDIR folder:

    ```
    mkdir -p asm-upgrade-tutorial && cd asm-upgrade-tutorial
    export WORKDIR=`pwd`
    ```

1. Create a KUBECONFIG file for this tutorial:

    ```
    touch ${WORKDIR}/asm-kubeconfig && export \
    KUBECONFIG=${WORKDIR}/asm-kubeconfig
    ```
 
## Preparing Terraform

1. Create a Google Cloud Platform Service account and give it the following roles:

    ```
    gcloud --project=${PROJECT_ID} iam service-accounts create \
      ${TERRAFORM_SA} --description="terraform-sa" \
      --display-name=${TERRAFORM_SA}

    ROLES=(
      'roles/servicemanagement.admin' \
      'roles/storage.admin' \
      'roles/serviceusage.serviceUsageAdmin' \
      'roles/meshconfig.admin' \
      'roles/compute.admin' \
      'roles/container.admin' \
      'roles/resourcemanager.projectIamAdmin' \
      'roles/iam.serviceAccountAdmin' \
      'roles/iam.serviceAccountUser' \
      'roles/iam.serviceAccountKeyAdmin' \
      'roles/gkehub.admin')
    for role in "${ROLES[@]}"
    do
      gcloud projects add-iam-policy-binding ${PROJECT_ID} \
      --member "serviceAccount:${TERRAFORM_SA}@${PROJECT_ID}.iam.gserviceaccount.com" \
      --role="$role"
    done
    ```

1. Create the Service Account credential JSON key for Terraform:

    ```
    gcloud iam service-accounts keys create \
      ${WORKDIR}/${TERRAFORM_SA}.json \
      --iam-account=${TERRAFORM_SA}@${PROJECT_ID}.iam.gserviceaccount.com
    ```

1. Set the Terraform credentials and project ID:

    ```
    export GOOGLE_APPLICATION_CREDENTIALS=${WORKDIR}/${TERRAFORM_SA}.json
    export TF_VAR_project_id=${PROJECT_ID}
    export TF_VAR_project_number=${PROJECT_NUM}
    ```

1. Create a Google Cloud Storage bucket and the backend resource for the Terraform state file:

    ```
    gsutil mb -p ${PROJECT_ID} gs://${PROJECT_ID}
    gsutil versioning set on gs://${PROJECT_ID}

    cat <<'EOF' > ${WORKDIR}/backend.tf_tmpl
    terraform {
     backend "gcs" {
       bucket  = "${PROJECT_ID}"
       prefix  = "tfstate"
     }
    }
    EOF

    envsubst < ${WORKDIR}/backend.tf_tmpl > ${WORKDIR}/backend.tf
    ```

## Installing Anthos Service Mesh

1. Create `main.tf` and `variables.tf` files to deploy a VPC, GKE cluster and Anthos Service Mesh:

    ```
    cat <<'EOF' > main.tf_tmpl
    data "google_client_config" "default" {}

    provider "kubernetes" {
      host                   = "https://${module.gke.endpoint}"
      token                  = data.google_client_config.default.access_token
      cluster_ca_certificate = base64decode(module.gke.ca_certificate)
    }

    module "vpc" {
      source  = "terraform-google-modules/network/google"
      version = "~> 3.0"

      project_id   = var.project_id
      network_name = var.network
      routing_mode = "GLOBAL"

      subnets = [
        {
          subnet_name   = var.subnetwork
          subnet_ip     = var.subnetwork_ip_range
          subnet_region = var.region
        }
      ]

      secondary_ranges = {
        (var.subnetwork) = [
          {
            range_name    = var.ip_range_pods
            ip_cidr_range = var.ip_range_pods_cidr
          },
          {
            range_name    = var.ip_range_services
            ip_cidr_range = var.ip_range_services_cidr
          }
        ]
      }
    }

    module "gke" {
      source                  = "terraform-google-modules/kubernetes-engine/google"
      project_id              = var.project_id
      name                    = var.cluster_name
      regional                = false
      region                  = var.region
      zones                   = var.zones
      release_channel         = "REGULAR"
      network                 = module.vpc.network_name
      subnetwork              = module.vpc.subnets_names[0]
      ip_range_pods           = var.ip_range_pods
      ip_range_services       = var.ip_range_services
      network_policy          = false
      identity_namespace      = "enabled"
      cluster_resource_labels = { "mesh_id" : "proj-${var.project_number}" }
      node_pools = [
        {
          name         = "asm-node-pool"
          autoscaling  = false
          auto_upgrade = true
          # ASM requires minimum 4 nodes and e2-standard-4
          node_count   = 2
          machine_type = "e2-standard-4"
        },
      ]
     }

    module "asm" {
      source = "github.com/terraform-google-modules/terraform-google-kubernetes-engine//modules/asm"
      cluster_name          = module.gke.name
      cluster_endpoint      = module.gke.endpoint
      project_id            = var.project_id
      location              = module.gke.location
      enable_all            = true
      asm_version           = var.asm_version
      managed_control_plane = false
      service_account       = "${TERRAFORM_SA}@${PROJECT_ID}.iam.gserviceaccount.com"
      key_file              = "${TERRAFORM_SA}.json"
      options               = ["envoy-access-log"]
      skip_validation       = false
      outdir                = "./${module.gke.name}-outdir-${var.asm_version}"
    }
    EOF

    cat <<'EOF' > variables.tf_tmpl
    variable "project_id" {}

    variable "project_number" {}

    variable "cluster_name" {
      default = "gke-central"
    }

    variable "region" {
      default = "us-central1"
    }

    variable "zones" {
      default = ["us-central1-a"]
    }

    variable "network" {
      default = "asm-vpc"
    }

    variable "subnetwork" {
      default = "subnet-01"
    }

    variable "subnetwork_ip_range" {
      default = "10.10.10.0/24"
    }

    variable "ip_range_pods" {
      default = "subnet-01-pods"
    }

    variable "ip_range_pods_cidr" {
      default = "10.100.0.0/16"
    }

    variable "ip_range_services" {
      default = "subnet-01-services"
    }

    variable "ip_range_services_cidr" {
      default = "10.101.0.0/16"
    }

    variable "asm_version" {
      default = "$ASM_MAJOR_VERSION"
    }
    EOF

    cat <<'EOF' > output.tf
    output "kubernetes_endpoint" {
      sensitive = true
      value     = module.gke.endpoint
    }

    output "client_token" {
      sensitive = true
      value     = base64encode(data.google_client_config.default.access_token)
    }

    output "ca_certificate" {
      sensitive = true
      value = module.gke.ca_certificate
    }

    output "service_account" {
      description = "The default service account used for running nodes."
      value       = module.gke.service_account
    }
    EOF

    envsubst < main.tf_tmpl > main.tf
    envsubst < variables.tf_tmpl > variables.tf
    ```

1. Initialize and apply Terraform:

    ```
    terraform init
    terraform plan
    terraform apply -auto-approve
    ```

## Accessing your cluster

1. Connect to the GKE cluster:

    ```
    gcloud container clusters get-credentials ${CLUSTER_1} --zone \
    ${CLUSTER_1_ZONE}
    ```

   Remember to unset your `KUBECONFIG` var when you're finished.


## Deploying Online boutique app

1. Get the ASM Revision number to label the namespace for automatic ASM proxy sidecar injection.

    ```
    export ASM_REV=$(kubectl --context=${CLUSTER_1_CTX} get pod -n istio-system -l app=istiod \
      -o jsonpath='{.items[0].metadata.labels.istio\.io/rev}')
    ```
3. Deploy Online Boutique to the GKE cluster:

    ```
    kpt pkg get \
    https://github.com/GoogleCloudPlatform/microservices-demo.git/release \
    online-boutique

    kubectl --context=${CLUSTER_1_CTX} create namespace online-boutique
    kubectl --context=${CLUSTER_1_CTX} label namespace online-boutique istio.io/rev=${ASM_REV}

    kubectl --context=${CLUSTER_1_CTX} -n online-boutique apply -f online-boutique
    ```

1. Wait until all Deployments are Ready:

    ```
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique wait --for=condition=available --timeout=5m deployment adservice
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique wait --for=condition=available --timeout=5m deployment checkoutservice
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique wait --for=condition=available --timeout=5m deployment currencyservice
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique wait --for=condition=available --timeout=5m deployment emailservice
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique wait --for=condition=available --timeout=5m deployment frontend
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique wait --for=condition=available --timeout=5m deployment paymentservice
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique wait --for=condition=available --timeout=5m deployment productcatalogservice
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique wait --for=condition=available --timeout=5m deployment shippingservice
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique wait --for=condition=available --timeout=5m deployment cartservice
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique wait --for=condition=available --timeout=5m deployment loadgenerator
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique wait --for=condition=available --timeout=5m deployment recommendationservice
    ```

### Accessing Online Boutique

Access the application using the `istio-ingressgateway` Service hostname:

```
kubectl --context=${CLUSTER_1_CTX} -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

## Upgrading Anthos Service Mesh

The `revisioned-istio-ingressgateway` is very important. This option creates a revisioned `istio-ingressgateway` Deployment which allows you to control when you switch to the new version. If you don't include this option, the script does an [in-place upgrade](https://istio.io/latest/docs/setup/upgrade/in-place/) of the `istio-ingressgateway`.

**Important**: You must also include any custom overlays or options you applied during the initial installation.

### Preparing Terraform to upgrade Anthos Service Mesh

1. Update the `variables.tf` file to the new version of Anthos Service Mesh:

   **Warning**: Include the `revisioned-istio-ingressgateway` to perform a revision-based upgrade. If you don't include this option, the script performs an [in-place upgrade](https://istio.io/latest/docs/setup/upgrade/in-place/) which entirely replaces the control plane components.

    ```
    sed -i s/ASM_MAJOR_VERSION/ASM_MAJOR_VERSION_UPGRADE/  \
      variables.tf_tmpl
    envsubst < variables.tf_tmpl > variables.tf
    sed -i 's/\benvoy-access-log\b/&,revisioned-istio-ingressgateway/' main.tf
    sed -i "/module.gke.location/a mode = \"upgrade\"" main.tf
    ```

1. Apply the terraform:

    ```
    terraform plan
    terraform apply -auto-approve
    ```

### Verifying the installed Anthos Service Mesh versions

1. Note the revision label that is on `istiod` and the `istio-ingressgateway`:

    ```
    kubectl --context=${CLUSTER_1_CTX} get pod -n istio-system -L istio.io/rev
    ```

   The output is similar to:

    ```
    NAME                                              READY   STATUS    RESTARTS   AGE     REV
    istio-ingressgateway-64457f47f8-vctxz             1/1     Running   0          7m53s   asm-196-2
    istio-ingressgateway-64457f47f8-vfwbv             1/1     Running   0          8m9s    asm-196-2
    istio-ingressgateway-asm-1102-3-5754c948b-crjkb   1/1     Running   0          52s     asm-1102-3
    istio-ingressgateway-asm-1102-3-5754c948b-rdkjz   1/1     Running   0          36s     asm-1102-3
    istiod-asm-1102-3-767855fcb5-8xp9s                1/1     Running   0          62s     asm-1102-3
    istiod-asm-1102-3-767855fcb5-cktpz                1/1     Running   0          62s     asm-1102-3
    istiod-asm-196-2-8544879bbd-bfnd6                 1/1     Running   0          8m20s   asm-196-2
    istiod-asm-196-2-8544879bbd-gp25n                 1/1     Running   0          8m20s   asm-196-2    
    ```

1. Get the upgraded ASM revision label.

    ```
    export ISTIOD_UPGRADED_POD=$(kubectl --context=${CLUSTER_1_CTX} get pod -n istio-system -l app=istiod | grep asm-110 | awk 'NR==1 {print $1}')
    export ASM_REV_UPGRADE=$(kubectl --context=${CLUSTER_1_CTX} -n istio-system get pod ${ISTIOD_UPGRADED_POD} \
      -o jsonpath='{.metadata.labels.istio\.io/rev}')
    echo $ASM_REV_UPGRADE
    ```

1. Switch the `istio-ingressgateway` to the new revision:

    ```
    kubectl --context=${CLUSTER_1_CTX} patch service -n istio-system istio-ingressgateway \
        --type='json' -p="[{"op": "replace", "path": "/spec/selector/service.istio.io~1canonical-revision", "value": "${ASM_REV_UPGRADE}"}]"
    ```

1. Add the revision label to the `online-boutique` namespace and remove the `istio-injection` label (if it exists):

    ```
    kubectl --context=${CLUSTER_1_CTX} label namespace online-boutique istio.io/rev=${ASM_REV_UPGRADE} istio-injection- --overwrite
    ```

   **Note**: You can safely ignore "istio-injection not found" in the output. If
   you see that message, it means that the namespace didn't previously have the `istio-injection` label.

1. Restart the Pods to trigger re-injection:

    ```
    kubectl --context=${CLUSTER_1_CTX} rollout restart deployment -n online-boutique
    ```

1. Wait until all Deployments are restarted.

    ```
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique rollout status -w --timeout=5m deployment adservice 
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique rollout status -w --timeout=5m deployment checkoutservice
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique rollout status -w --timeout=5m deployment currencyservice
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique rollout status -w --timeout=5m deployment emailservice
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique rollout status -w --timeout=5m deployment frontend
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique rollout status -w --timeout=5m deployment paymentservice
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique rollout status -w --timeout=5m deployment productcatalogservice
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique rollout status -w --timeout=5m deployment shippingservice
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique rollout status -w --timeout=5m deployment cartservice
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique rollout status -w --timeout=5m deployment loadgenerator
    kubectl --context=${CLUSTER_1_CTX} -n online-boutique rollout status -w --timeout=5m deployment recommendationservice
    ```

    The output is similar to the following:
    
    ```
    deployment "adservice" successfully rolled out
    deployment "checkoutservice" successfully rolled out
    deployment "currencyservice" successfully rolled out
    deployment "emailservice" successfully rolled out
    deployment "frontend" successfully rolled out
    deployment "paymentservice" successfully rolled out
    deployment "productcatalogservice" successfully rolled out
    deployment "shippingservice" successfully rolled out
    deployment "cartservice" successfully rolled out
    deployment "loadgenerator" successfully rolled out
    deployment "recommendationservice" successfully rolled out
    ```

1. Verify that your Pods are configured to point to the new version of `istiod`:

    ```
    kubectl --context=${CLUSTER_1_CTX} get pods -n online-boutique -l istio.io/rev=${ASM_REV_UPGRADE}
    ```

   You should see all the Pods for online boutique. You can also verify that the workloads are working correctly by revisiting the Online Botique app. From here you have two options:

   1. Follow the steps in **Complete the upgrade** to transition to the new
      version of `istiod` if everything working as expected.
   1. Follow the steps in **Rollback the upgrade** if there's any issue.

### Finalizing the upgrade

In this section, you finalize the ASM upgrade and delete the previous ASM version artifacts. Note that you cannot rollback to the previous ASM version once you perform the steps in this section. If you would like to rollback to the previous ASM version, skip this section and proceed to the next section.

1. Configure the validating webhook to use the new control plane:

    ```
    curl https://raw.githubusercontent.com/GoogleCloudPlatform/anthos-service-mesh-packages/1.9.5-asm.2%2Bconfig1/asm/istio/istiod-service.yaml > istiod-service.yaml_tmpl
    sed -e s/ASM_REV/${ASM_REV_UPGRADE}/ \
       istiod-service.yaml_tmpl > istiod-service.yaml
    kubectl --context=${CLUSTER_1_CTX} apply -f istiod-service.yaml
    ```

1. Delete the old `istio-ingressgateway` Deployment:

    ```
    kubectl --context=${CLUSTER_1_CTX} delete deploy -l \
       app=istio-ingressgateway,istio.io/rev=${ASM_REV} -n istio-system \
       --ignore-not-found=true
    ```

1. Delete the old version of `istiod`:

    ```
    kubectl --context=${CLUSTER_1_CTX} delete \
      Service,Deployment,HorizontalPodAutoscaler,PodDisruptionBudget \
      istiod-${ASM_REV} -n istio-system --ignore-not-found=true
    ```

1. Remove the old version of the `IstioOperator` configuration:

    ```
    kubectl --context=${CLUSTER_1_CTX} delete IstioOperator installed-state-${ASM_REV} \
       -n istio-system
    ```

### Rolling back to previous version

In this section, you rollback to the previous ASM version. If you have already performed the steps in the `Finalizing your upgrade` section, then you cannot rollback.

1. Restart the Pods to trigger re-injection so the proxies have the previous
   version:

    ```
    kubectl --context=${CLUSTER_1_CTX} rollout restart deployment -n online-boutique
    ```

1. Remove the new istio-ingressgateway Deployment:

    ```
    kubectl --context=${CLUSTER_1_CTX} delete deploy -l \
      app=istio-ingressgateway,istio.io/rev=$ASM_REV_UPGRADE -n istio-system \
      --ignore-not-found=true
    ```

1. Remove the new version of `istiod`:

    ```
    kubectl --context=${CLUSTER_1_CTX} delete \
      Service,Deployment,HorizontalPodAutoscaler,PodDisruptionBudget \
      istiod-${ASM_REV_UPGRADE} -n istio-system --ignore-not-found=true
    ```

1. Remove the new version of the `IstioOperator` configuration:

    ```
    kubectl --context=${CLUSTER_1_CTX} delete IstioOperator \
      installed-state-${ASM_REV_UPGRADE} -n istio-system
    ```
   The expected output is similar to:

    ```
    istiooperator.install.istio.io "installed-state-REVISION" deleted
    ```

1. Revert Terraform values to previous version of Anthos Service Mesh (1.8):

    ```
    sed -i s/ASM_MAJOR_VERSION_UPGRADE/ASM_MAJOR_VERSION/ variables.tf_tmpl
    envsubst < variables.tf_tmpl > variables.tf
    sed -i 's/,revisioned-istio-ingressgateway//' main.tf
    sed -i 's/mode = \"upgrade\"//' main.tf
    ```

## Clean up

### Terraform destroy

Use the `terraform destory` command to destroy all Terraform resources:

```
cd ${WORKDIR}
terraform destroy -auto-approve
```

### Delete the project

Alternatively, you can delete the project.

Deleting a project has the following consequences:

- If you used an existing project, you'll also delete any other work that you've done in the project.
- You can't reuse the project ID of a deleted project. If you created a custom project ID that you plan to use in the future, delete the resources inside the project instead. This ensures that URLs that use the project ID, such as an `appspot.com` URL, remain available.

To delete a project, do the following:

1.  In the Cloud Console, go to the [Projects page](https://console.cloud.google.com/iam-admin/projects).
1.  In the project list, select the project you want to delete and click **Delete**.
1.  In the dialog, type the project ID, and then click **Shut down** to delete the project.

## What's next

- Learn more about [community support for Terraform](https://cloud.google.com/docs/terraform#terraform_support_for)
- Learn more about [Anthos Service Mesh](https://cloud.google.com/service-mesh)