# Operator Lifecycle Manager

The [Operator Lifecycle Manager (OLM)][olm] provides a declarative way to install and upgrade operators and their
dependencies.

You can install the Trivy Operator from [OperatorHub.io] or [ArtifactHUB] by creating the OperatorGroup, which
defines the Operator's multitenancy, and Subscription that links everything together to run the operator's pod.

As an example, let's install the Operator from the OperatorHub catalog in the `trivy-system` namespace and
configure it to watch the `default` namespaces:

1. Install the Operator Lifecycle Manager:
   ```
   curl -L https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.20.0/install.sh -o install.sh
   chmod +x install.sh
   ./install.sh v0.20.0
   ```

2. Create the namespace to install the operator in:
   ```
   kubectl create ns trivy-system
   ```
3. Create the OperatorGroup to select all namespaces:
   ```
   cat << EOF | kubectl apply -f -
   apiVersion: operators.coreos.com/v1
   kind: OperatorGroup
   metadata:
     name: trivy-operator-group
     namespace: trivy-system
   EOF
   ```
4. Install the operator by creating the Subscription:
   ```
   cat << EOF | kubectl apply -f -
   apiVersion: operators.coreos.com/v1alpha1
   kind: Subscription
   metadata:
     name: trivy-operator-subscription
     namespace: trivy-system
   spec:
     channel: alpha
     name: trivy-operator
     source: operatorhubio-catalog
     sourceNamespace: olm
     installPlanApproval: Automatic
     config:
       env:
       - name: OPERATOR_EXCLUDE_NAMESPACES
         value: "kube-system"
   EOF
   ```
   The operator will be installed in the `trivy-system` namespace and will select all namespaces, except
   `kube-system` and `trivy-system`. 

5. After install, watch the operator come up using the following command:
   ```console
   $ kubectl get clusterserviceversions -n trivy-system
   NAME                        DISPLAY              VERSION   REPLACES                     PHASE
   trivy-operator.{{ git.tag }}  Trivy Operator   {{ git.tag[1:] }}    trivy-operator.{{ var.prev_git_tag }}   Succeeded
   ```
   If the above command succeeds and the ClusterServiceVersion has transitioned from `Installing` to `Succeeded` phase
   you will also find the operator's Deployment in the same namespace where the Subscription is:
   ```console
   $ kubectl get deployments -n trivy-system
   NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
   trivy-operator   1/1     1            1           11m
   ```
   If for some reason it's not ready yet, check the logs of the Deployment for errors:
   ```
   kubectl logs deployment/trivy-operator -n trivy-system
   ```

## Uninstall

To uninstall the operator delete the Subscription, the ClusterServiceVersion, and the OperatorGroup:

```
kubectl delete subscription trivy-operator-subscription -n trivy-system
kubectl delete clusterserviceversion trivy-operator.{{ git.tag }} -n trivy-system
kubectl delete operatorgroup trivy-operator-group -n trivy-system
kubectl delete ns trivy-system
```

You have to manually delete custom resource definitions created by the OLM operator:

!!! danger
    Deleting custom resource definitions will also delete all security reports generated by the operator.

    ```
    kubectl delete crd vulnerabilityreports.aquasecurity.github.io
    kubectl delete crd configauditreports.aquasecurity.github.io
    kubectl delete crd clusterconfigauditreports.aquasecurity.github.io
    kubectl delete crd exposedsecrets.aquasecurity.github.io
    ```

[olm]: https://github.com/operator-framework/operator-lifecycle-manager/
[OperatorHub.io]: https://operatorhub.io/operator/trivy-operator/
[ArtifactHUB]: https://artifacthub.io/
