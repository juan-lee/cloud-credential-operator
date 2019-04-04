# cloud-credential-operator <-> azure integration

## openshift/installer ✅
openshift/installer is responsible for creating and copying the root azure credentials to the openshift cluster.

The credentials will be exposed as a secret called `azure-creds` in the `kube-system` namespace with the following format.

``` yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  annotations:
    cloudcredential.openshift.io/mode: mint
  name: azure-creds
  namespace: kube-system
data:
  azure_client_id: ****
  azure_client_secret: ****
```

This has already been implemented and has a pending PR.

## Passthrough mode
Passthrough mode will simply reuse the root azure credentials (service principal) for each `CredentialsRequest`.

## Mint mode
Mint mode allows for operators to request a credential with least privileged access to get its job done. For example, the ingress operator can request a credential that only has access to administer azure load balancer resources. In azure this means that the service principal with the proper AAD tenant permissions, or root credential, would create a new service principal and assign the `Network Contributor` role so the ingress operator can manage load balancers and nothing else.

### Proposed API

#### types
``` golang
// AzureProviderSpec contains the required information to create RBAC policies in Azure.
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
type AzureProviderSpec struct {
	metav1.TypeMeta `json:",inline"`

	// RoleBindings contains a list of roles that should be associated with the minted credential.
	RoleBindings []RoleBinding `json:"roleBindings"`
}

// RoleBinding models the Azure RBAC Role Binding
type RoleBinding struct {
	// Role defines a set of permissions that should be associated with the minted credential.
	Role string `json:"role"`

	// Scope specifies the resource(s) this binding should apply to. (or "*" for the azure subscription)
	Scope string `json:"scope"`
}

// AzureProviderStatus contains the status of the credentials request in Azure.
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
type AzureProviderStatus struct {
	metav1.TypeMeta `json:",inline"`

	// ClientID is the ID of the User created in Azure for these credentials.
	ClientID string `json:"clientID"`
}
```

#### example manifest
``` yaml
apiVersion: cloudcredential.openshift.io/v1beta1
kind: CredentialsRequest
metadata:
  name: openshift-ingress
  namespace: openshift-cloud-credential-operator
spec:
  providerSpec:
    apiVersion: cloudcredential.openshift.io/v1
    kind: AzureProviderSpec
    roleBindings:
    - role: 'Network Contributor'
      scope: '/subscriptions/<subscription_id>/resourceGroups/<resourcegroup_name>/providers/Microsoft.Network/loadBalancers/<loadbalancer_name>'
```

### Azure APIs Needed by cloud-credential-operator
#### Mint Credential
https://github.com/Azure/azure-sdk-for-go/blob/master/services/graphrbac/1.6/graphrbac/serviceprincipals.go#L48

#### Role Assignment
https://github.com/Azure/azure-sdk-for-go/blob/master/services/authorization/mgmt/2015-07-01/authorization/roleassignments.go#L56

##### Sample
https://github.com/Azure-Samples/azure-sdk-for-go-samples/blob/master/authorization/authorization.go

### Root Azure Credentials
In order for the cloud-credential-operator to mint credentials with least privileged access the Azure Active Directory tenant administrator must grant the root service principal `Windows Active Directory` > `Read and write all applications` application permissions. Otherwise the root credential (service principal) will not be able to create credentials. If the tenant admin is not willing to grant these permissions then the user will need to fall back to Passthrough mode.

## Glossary
- [Service Principal](https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals)
- [Azure RBAC](https://docs.microsoft.com/en-us/azure/role-based-access-control/overview)
