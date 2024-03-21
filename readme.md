# Creating custom role on AAD protected AKS 

This repo demostrates how to create read only privileges against the cluster to a particular set of API resources in AKS protected by Azure AAD. These APIs are often restricted to only Cluster Admin. The background is to grant restricted readonly escalated privileges to the k8s users. for example 
 * View quota
 * View nodes and metrics 
 * Events

## Concept 

Essentially the process would involve allowing to the role read everything in the **DataActions** and then explicitiy denying other actions within the **notDataActions** group. 
This approach is required due to the usage of CRDs and it is currently not supported to explicity deny or allow access to these types.

 While its possible to grant read/delete/write rights to the CRD defintions. So a user can list and delete and create these but we cannot limit of specific types of CRDs. For example the Gatekeeper defintions like ConstraintTemplates, Constraints. Similar this applies tothings like Cert manager objects 

In the example below, we grant **only read** across all API types then selectively remove permissions. This approach is required to accomodate the lack of RBAC across the CRD types. 
* This results in all types been readonly by default 
* NotDataActions then restricts the APIs we don't want accessed

```
permissions {
  actions = [ ]
    data_actions = [
    ## Granted read access to all resources
    "Microsoft.ContainerService/managedClusters/*/read" 
  ]
  not_actions = [ ]
  not_data_actions = [ 
    # Insert 10's or 100's of rows here to remove permissions that we didn't want to give
  ]
}
```

<mark>Recommendation:</mark> The above role concept should be used in combination with either a namespaced role. Such as RBAC Writer, RBAC Reader, RBAC Admin. 

Azure RBAC is an additive model, so your effective permissions are the sum of your role assignments.

### Apply on Namespace scope
Applying the  above role to a namespace level will restrict a user to only that namespace but concurrently grant access to non-namespaced objects 
```
az role assignment create --role "Azure Kubernetes Service RBAC Reader" --assignee <AAD-ENTITY-ID> --scope $AKS_ID/namespaces/<namespace-name>
```
## Deploy Custom role 
Theres are thress definitions files that can be deploy   
 * [ ClusterScopedReadonlyClusterAdmin ](ClusterScopedReadonlyClusterAdmin.json) Should be used in combination with one of the builtin roles. when used with a built inrole this gives a normal admin on a namespace read access to selected non-namespaced objects. 
 * [ ClusterScopedReadonlyNodes ](ClusterScopedReadonlyNodes.json): Should be used in combination with one of the builtin roles. when used with a built inrole this gives a normal admin on a namespace read access to selected non-namespaced objects.  This differs from the [ ClusterScopedReadonlyClusterAdmin ](ClusterScopedReadonlyClusterAdmin.json)as it does not give wildcard read access and therefore does not allow a user to view CRDs. For readonly access against nodes, then this is recommended. 
 * [limitedAdminReader](limitedAdminReader.json) : Includes the wildcard readme across all resources but explicity add read and write access for other API types. the assumption with this roles is the user does not have any additional roles.  This was used as a test Role and it is instead recommended to the ClusterScopedReadonlyClusterAdmin.json role 

<mark>Note:</mark> three definitions restrict access to roles and secrets. 

To deploy execute the following. 
```
az role definition create --role-definition  @ClusterScopedReadonlyClusterAdmin.json
```
<mark>Note:</mark>
Before deploying update the field `AssignableScopes`to include either the root `/`, subscription `/subscriptions/XXXXXXXX` or resource group `/subscriptions/XXXXXXXX/resourcegroups/XXXXXXX`
## Delete role

 ```
 az role definition delete -n "Azure Kubernetes Service RBAC ReadOnly Admin"
 ```

### Details 
The following outlines why each right was applied and how. 

* Grant read to all objects, including non-namespaced apis. The reason a wildcard is used is to ensure the CRD definitions are included
  *   `Microsoft.ContainerService/managedClusters/*/read`   
<mark>this is not used in  ClusterScopedReadonlyNodes , and example can be found in [text](ClusterScopedReadonlyClusterAdmin.json) </mark> 
* Grant read access to node and non-namespace level objects. (although already included above )
```           
    "DataActions": [
        "Microsoft.ContainerService/managedClusters/serviceaccounts/read",
        "Microsoft.ContainerService/managedClusters/node.k8s.io/runtimeclasses/read",
        "Microsoft.ContainerService/managedClusters/nodes/read",
        "Microsoft.ContainerService/managedClusters/apps/controllerrevisions/read",
        "Microsoft.ContainerService/managedClusters/events.k8s.io/events/read",
        "Microsoft.ContainerService/managedClusters/limitranges/read",
        "Microsoft.ContainerService/managedClusters/metrics.k8s.io/pods/read",
        "Microsoft.ContainerService/managedClusters/metrics.k8s.io/nodes/read",
        "Microsoft.ContainerService/managedClusters/resourcequotas/read",
        "Microsoft.ContainerService/managedClusters/namespaces/read"

    ],
```

* The role has complete Read access across all APIs via the wildcard DataActions entry . Using NotDataActions we can selectively disable access across certain APIs. 

    
```
"NotDataActions": [
        "Microsoft.ContainerService/managedClusters/admissionregistration.k8s.io/*",
        "Microsoft.ContainerService/managedClusters/api/*",
        "Microsoft.ContainerService/managedClusters/apiregistration.k8s.io/apiservices/*",
        "Microsoft.ContainerService/managedClusters/apis/*",
        "Microsoft.ContainerService/managedClusters/authentication.k8s.io/*",
        "Microsoft.ContainerService/managedClusters/authorization.k8s.io/*",
        "Microsoft.ContainerService/managedClusters/bindings/*",
        "Microsoft.ContainerService/managedClusters/certificates.k8s.io/*",
        "Microsoft.ContainerService/managedClusters/componentstatuses/*",
        "Microsoft.ContainerService/managedClusters/coordination.k8s.io/*",
        "Microsoft.ContainerService/managedClusters/discovery.k8s.io/endpointslices/*",
        "Microsoft.ContainerService/managedClusters/endpoints/*",
        "Microsoft.ContainerService/managedClusters/extensions/podsecuritypolicies/*",
        "Microsoft.ContainerService/managedClusters/flowcontrol.apiserver.k8s.io/*",
        "Microsoft.ContainerService/managedClusters/groups/impersonate/*",
        "Microsoft.ContainerService/managedClusters/healthz/*",
        "Microsoft.ContainerService/managedClusters/livez/*",
        "Microsoft.ContainerService/managedClusters/metrics/*",
        "Microsoft.ContainerService/managedClusters/metrics.k8s.io/*",
        "Microsoft.ContainerService/managedClusters/networking.k8s.io/ingressclasses/*",
        "Microsoft.ContainerService/managedClusters/node.k8s.io/*",
        "Microsoft.ContainerService/managedClusters/nodes/write",
        "Microsoft.ContainerService/managedClusters/nodes/delete",
        "Microsoft.ContainerService/managedClusters/openapi/v2/*",
        "Microsoft.ContainerService/managedClusters/persistentvolumes/*",
        "Microsoft.ContainerService/managedClusters/podtemplates/*",
        "Microsoft.ContainerService/managedClusters/policy/podsecuritypolicies/*",
        "Microsoft.ContainerService/managedClusters/rbac.authorization.k8s.io/*",
        "Microsoft.ContainerService/managedClusters/readyz/*",
        "Microsoft.ContainerService/managedClusters/resetMetrics/*",
        "Microsoft.ContainerService/managedClusters/scheduling.k8s.io/*",
        "Microsoft.ContainerService/managedClusters/secrets/*",
        "Microsoft.ContainerService/managedClusters/storage.k8s.io/*",
        "Microsoft.ContainerService/managedClusters/swagger-api/*",
        "Microsoft.ContainerService/managedClusters/ui/*",
        "Microsoft.ContainerService/managedClusters/users/*",
        "Microsoft.ContainerService/managedClusters/version/*"
    ]
```

## Issues with CRDs

 As documented here https://github.com/Azure/AKS/issues/2896 .  Its currently not possible to control access to CRD defintions. Its for this reason the wildcard read only approach is used to ensure users can still read the CRDs. Then NotDataActions deny specific APIs that we don't want the user to access.  


## References 
### Additive Permissions in Azure 
Azure Resource Manager determines if the action in the API call is included in the roles the user has for this resource. If the roles include Actions that have a wildcard (*), the effective permissions are computed by subtracting the NotActions from the allowed Actions. Similarly, the same subtraction is done for any data actions.
* `Actions - NotActions = Effective management permissions`

* `DataActions - NotDataActions = Effective data permissions`

 * https://learn.microsoft.com/en-us/azure/role-based-access-control/overview#how-azure-rbac-determines-if-a-user-has-access-to-a-resource

### Github AKS project 
* https://github.com/Azure/AKS/issues/2896 
### How to view existing AKS RBAC  roles 

**View Namespace level reader. when applied to namespace**
* `az role definition list  --name "Azure Kubernetes Service RBAC Reader"`

**View Namespace level admin . when applied to namespace**
* `az role definition list  --name "Azure Kubernetes Service RBAC Admin"`

**Allows modification of all objs in namespace but not modifiying roles**
* `az role definition list  --name "Azure Kubernetes Service RBAC Writer"`

