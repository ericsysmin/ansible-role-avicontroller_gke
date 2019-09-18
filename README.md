# avinetworks.avicontroller_gke

Ansible role that deploys an Avi Controller on Google GKE. We will deploy only
one controller cluster with either 1 or 3 controllers per namespace.

>  **Warning:**
>
> -   Role will override any existing configuration (if a value is different
>     here than in the k8s config it will be overriden)

## Requirements

-   GKE Cluster
-   GKE Node Pool w/Labels if using affinity or nodeSelector

### OS Packages Required

-   python >= 2.7

### Python Libraries Required

-   openshift >= 0.6
-   PyYAML >= 3.11
-   requests >= 2.18.4
-   google-auth >= 1.3.0

## Required Environment Variables

When using GKE and K8s we discovered that some variables need defined at the
environment level as to handle the proper authentication to GKE.

```bash
K8S_AUTH_KUBECONFIG=/location/of/.kubeconfig
# GOOGLE_APPLICATION_CREDENTIALS is used when trying to auth to K8s, I could not
# find an alternative way to provide this and it work
GOOGLE_APPLICATION_CREDENTIALS=/location/of/service_account_file.json
```

## Required Steps

1.  You will need to set your current cluster via `gcloud` to configure the
    proper .kube/config data. To do so run the following command

    ```bash
    gcloud container clusters get-credentials <cluster-name> --region=<region> --zone=<zone>
    ```

2.  Verify that you are in the correct context by typing
    ```bash
    kubectl config current-context
    ```
    It should return something in this format.
    ```text
    gke_{{ gke_project }}_{{ region }}_{{ cluster_name }}
    ```

## Role Variables

A description of the settable variables for this role should go here, including
any variables that are in defaults/main.yml, vars/main.yml, and any variables
that can/should be set via parameters to the role. Any variables that are read
from other roles and/or the global scope (ie. hostvars, group vars, etc.) should
be mentioned here as well.

### Variables

| Variable                            | Required     | Default                                                   | Comments                                                                         |
| ----------------------------------- | ------------ | --------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `avi_namespace`                     | Yes          |                                                           | Namespace that the controller should be created in                               |
| `avi_controller_state`              | No           | `present`                                                 | Sets the state of the deployment. ex. `present`, `absent`, `suspended`, `resume` |
| `avi_force_state`                   | No           | `false`                                                   | Allows forcing to state of the deployment. Skips checks. `true`, `false`         |
| `avi_controller_count`              | No           | `1`                                                       | How many controllers we should create. ex. `1` or `3`                            |
| `avi_controller_prefix`             | No           | `avi-controller`                                          | Prefix that the name of the controller and assets should have                    |
| `avi_controller_username`           | Yes (absent) | `admin`                                                   | Only required when state is absent used for verifying no SE or Virtual Services  |
| `avi_controller_password`           | Yes (absent) | `None`                                                    | Only required when state is absent used for verifying no SE or Virtual Services  |
| `avi_gcp_region`                    | No           |                                                           | GCP region that we want to deploy the controller to                              |
| `avi_gcp_project`                   | No           |                                                           | GCP project that the controller should be deployed in                            |
| `avi_gcp_auth_kind`                 | No           |                                                           | Type of authentication to GCP to use                                             |
| `avi_gcp_service_account_file`      | No           |                                                           | Location of the service_account_file when using serviceaccount                   |
| `avi_k8s_auth_kubeconfig`           | No           | `{{ ansible_env.HOME }}/.kube/config`                     | Location of the kubeconfig that we will use                                      |
| `avi_controller_storage_class_name` | No           | `{{ avi_controller_prefix }}-regionalpd-storageclass-ssd` | Name of the storage class to be used by the controller disk                      |
| `avi_controller_disk_size`          | No           | `64`                                                      | Controller disk size in GB                                                       |
| `avi_controller_version`            | No           | `18.2.3-9063-20190501.224326`                             | Version of Avi that should be used on the pod                                    |
| `avi_controller_container_image`    | No           | `avinetworks/controller:{{ avi_controller_version }}`     | The image that will be used to create the controller pod                         |
| `avi_controller_namespace_labels`   | No           | `None`                                                    | K8s labels you'd like to attach to the namespace                                 |
| `avi_gcp_compute_addresses`         | No           | `Auto-generated`                                          | Array of compute addresses created by role for controllers                       |
| `avi_controller_affinity`           | No           | `None`                                                    | Sets the k8s affinity of the controller pod                                      |
| `avi_controller_nodeselector`       | No           | `None`                                                    | Sets the nodeSelector for the controller pod                                     |

### Advanced variables

_These values are not required and are advanced, they are not meant to be
updated unless required for specific overrides._

#### Default Variables

| Variable                              | Comments                            |
| ------------------------------------- | ----------------------------------- |
| `avi_controller_k8s_namespace`        | Namespace definition                |
| `avi_controller_k8s_external_service` | Ensures the external service exists |
| `avi_controller_k8s_service`          | Ensures the service exists          |
| `avi_controller_k8s_statefulset`      | Ensures the statefulset exists      |
| `avi_controller_k8s_storage_class`    | Ensures the StorageClass exists     |

## Dependencies

A list of other roles hosted on Galaxy should go here, plus any details in
regards to parameters that may need to be set for other roles, or variables that
are used from other roles.

## Usage

Please note when resuming a deployment, it is not different than present when making  
please use `deployment_state: present` when resuming a suspended deployment.

## Example Playbook

### Creating a controller cluster

Including an example of how to use your role (for instance, with variables
passed in as parameters) is always nice for users too:

```yaml
- hosts: servers
  roles:
    - role: avinetworks.avicontroller_gke
      avi_controller_count: 3
      avi_controller_version: 18.2.3-9063-20190501.224326
      avi_controller_prefix: deployment-address
      avi_gcp_project: my-project
      avi_gcp_region: us-west1
      avi_gcp_auth_kind: serviceaccount
      avi_gcp_service_account_file: ~/service_account_file.json
      avi_namespace: 26abc3b9d1fc4cfc8f42ad86d9606fb9
      avi_controller_disk_size: 64
      avi_controller_container_image: "gcr.io/{{ avi_gcp_project }}/controller:{{ avi_controller_version }}"
      avi_controller_storage_class_name: regionalpd-storageclass-ssd
      avi_controller_affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node_label
                    operator: In
                    values:
                      - label_value
```

```yaml
- hosts: servers
  roles:
    - role: avinetworks.avicontroller_gke
      avi_controller_count: 3
      avi_controller_version: 18.2.3-9063-20190501.224326
      avi_controller_prefix: deployment-address
      avi_gcp_project: my-project
      avi_gcp_region: us-west1
      avi_gcp_auth_kind: serviceaccount
      avi_gcp_service_account_file: ~/service_account_file.json
      avi_namespace: 26abc3b9d1fc4cfc8f42ad86d9606fb9
      avi_controller_disk_size: 64
      avi_controller_container_image: "gcr.io/{{ avi_gcp_project }}/controller:{{ avi_controller_version }}"
      avi_controller_storage_class_name: regionalpd-storageclass-ssd
      avi_controller_nodeselector:
        node_label: label_value
```

### Deleting a controller cluster

When you are deleting a controller cluster we check and make sure that you do not have any current VS's or Service engines
so that you do not end up with orphaned service engines.

```yaml
- hosts: servers
  roles:
    - role: avinetworks.avicontroller_gke
      avi_controller_state: absent
      avi_controller_count: 3
      avi_controller_version: 18.2.3-9063-20190501.224326
      avi_controller_prefix: deployment-address
      avi_gcp_project: my-project
      avi_gcp_region: us-west1
      avi_gcp_auth_kind: serviceaccount
      avi_gcp_service_account_file: ~/service_account_file.json
      avi_namespace: 26abc3b9d1fc4cfc8f42ad86d9606fb9
      avi_controller_disk_size: 64
      avi_controller_container_image: "gcr.io/{{ avi_gcp_project }}/controller:{{ avi_controller_version }}"
      avi_controller_storage_class_name: regionalpd-storageclass-ssd
      avi_controller_nodeselector:
        node_label: label_value
```

## License

Apache 2.0

## Author Information

Eric Anderson

[Avi Networks](http://avinetworks.com)
