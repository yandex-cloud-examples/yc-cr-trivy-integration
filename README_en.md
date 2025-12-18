# Integrating Starboard with Yandex Cloud Container Registry to scan running images

[Starboard](https://aquasecurity.github.io/starboard/v0.14.0/) is a great free tool you can use to automatically update security reports in response to workload and other changes on a Kubernetes cluster: for example, initiating a vulnerability scan when a new pod is started or running CIS Benchmarks when a new node is added.

Integrating Starboard with [Yandex Cloud Container Registry](https://yandex.cloud/en/docs/container-registry/) will allow you to automatically scan your images for vulnerabilities as new pods start.

Yandex Cloud Managed Service for Kubernetes uses a service account with the `container-registry.images.puller` role [assigned for a K8s node](https://yandex.cloud/en/docs/managed-kubernetes/security/#sa-annotation) for authentication with the Yandex Cloud Container Registry. However, Starboard employs its own authentication feature for private registries. 

Starboard can authenticate with various private container registries, as detailed in the [Private Registries](https://aquasecurity.github.io/starboard/v0.14.0/integrations/private-registries/) documentation. To achieve this, it simply copies the [k8s image pull secret](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/), a secret that contains authentication credentials and is assigned to pods for authenticating with registries.

To provide the Starboard Operator with the required secret, you can use [authentication in the registry with authorized keys](https://yandex.cloud/en/docs/container-registry/operations/authentication#sa-json) from a separate service account. 

To do this, follow these steps:

1. Create a service account [with the UI](https://yandex.cloud/en/docs/iam/operations/sa/create) or CLI:

    ```
    yc iam service-account create --name yc-cr-starboard
    ```

1. Assign the `container-registry.images.puller` role to the service account. You can [use the UI](https://yandex.cloud/en/docs/iam/operations/sa/assign-role-for-sa) or CLI:

    ```
    yc container registry add-access-binding \
      --service-account-name yc-cr-starboard \
      --role container-registry.images.puller
    ```

1. Create an authorized key for the service account and save it to a file. [Use the UI](https://yandex.cloud/en/docs/iam/operations/authorized-key/create) or CLI:

    ```
    yc iam key create --service-account-name yc-cr-starboard --output authorized-key.json
    ```

4. Create a K8s secret specifically for authentication using the [authorized key of the service account](https://yandex.cloud/en/docs/container-registry/operations/authentication#sa-json).

    ```
    cat <<EOF | kubectl apply -f -
    apiVersion: v1
    data:
      .dockerconfigjson: $(kubectl create secret docker-registry regcred --docker-server=cr.yandex --docker-username=json_key --docker-password="$(cat ./key.json)" --dry-run=client --output="jsonpath={.data.\.dockerconfigjson}" | base64 --decode | jq 'del(.auths."cr.yandex".auth)' | base64 )
    kind: Secret
    metadata:
      name: regcred
    type: kubernetes.io/dockerconfigjson
    EOF
    ```

    Details of the secret's format

    By default, creating a Docker secret according to [Create a Secret by providing credentials on the command line](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-secret-by-providing-credentials-on-the-command-line), e.g., by running this command:

    ```
    kubectl create secret docker-registry regcred --docker-server=cr.yandex --docker-username=json_key --docker-password="$(cat ./key.json)" --dry-run=client -o yaml
    ```

    returns a secret formatted as follows:

    ```
    apiVersion: v1
    data:
      .dockerconfigjson: {"auths":{"cr.yandex":{"username":"json_key","password":"something__","auth":"anNvbl9rZXk6ewogICAiaWQiOi..."}}}
    kind: Secret
    metadata:
      creationTimestamp: null
      name: regcred
    type: kubernetes.io/dockerconfigjson
    ```

    However, Starboard authentication requires the format *without the second `auth` field*, which is why the above command filters the latter out.

1. Assign the created secret for the loads that pull images from Yandex Cloud Container Registry, following [Create a Pod that uses your Secret](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-pod-that-uses-your-secret). You can also assign that secret to the `default` service account mapped to your pods, as per [Add ImagePullSecrets to a service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#add-imagepullsecrets-to-a-service-account).


1. Follow the [regular Starboard guide](https://aquasecurity.github.io/starboard/v0.14.0/operator/getting-started/) for installing, configuring, and employing the Starboard Operator. 

1. What you can do with Starboard scans:
- Manually analyze them by means of the [VulnerabilityReport](https://aquasecurity.github.io/starboard/v0.14.0/crds/vulnerability-report/) CRDs.
- Visualize them with [Octant or Lens](https://aquasecurity.github.io/starboard/v0.14.0/integrations/octant/).
- Automate reading VulnerabilityReport CRDs and sending them to your SIEM, e.g., [Yandex Managed Service for Elasticsearch](https://yandex.cloud/en/services/managed-elasticsearch).
- Analyze them on your Security Dashboard with [Cluster image scanning](https://docs.gitlab.com/ee/user/application_security/cluster_image_scanning/) in [Yandex Managed Service for GitLab](https://yandex.cloud/en/services/managed-gitlab). 
