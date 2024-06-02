# API Authorization

[TOC]
Kubernetes authorization of API requests takes place within the API server. The API server evaluates all of the request attributes against all policies, potentially also consulting external services, and then allows or denies the request.

All parts of an API request must be allowed by some authorization mechanism in order to proceed. In other words: access is denied by default.
## Concepts
* API groups
API groups make it easier to extend the Kubernetes API. The API group is specified in a REST path and in the apiVersion field of a serialized object.
There are several API groups in Kubernetes:
  * The core (also called legacy) group is found at REST path /api/v1. The core group is not specified as part of the apiVersion field, for example, apiVersion: v1.
  * The named groups are at REST path /apis/\$GROUP\_NAME/\$VERSION and use apiVersion: \$GROUP\_NAME/\$VERSION (for example, apiVersion: batch/v1). You can find the full list of supported API groups in Kubernetes API reference [^4].
* API verbs
Almost all object resource types support the standard HTTP verbs - GET, POST, PUT, PATCH, and DELETE. Kubernetes als`o uses its own verbs, which are often written lowercase to distinguish them from HTTP verbs.
Kubernetes uses the term list to describe returning a collection of resources to distinguish from retrieving a single resource which is usually called a get. If you sent an HTTP GET request with the ?watch query parameter, Kubernetes calls this a watch and not a get (see Efficient detection of changes for more details).
For PUT requests, Kubernetes internally classifies these as either create or update based on the state of the existing object. An update is different from a patch; the HTTP verb for a patch is PATCH.

* Resource URI
All resource types are either scoped by the cluster (/apis/GROUP/VERSION/*) or to a namespace (/apis/GROUP/VERSION/namespaces/NAMESPACE/*). A namespace-scoped resource type will be deleted when its namespace is deleted and access to that resource type is controlled by authorization checks on the namespace scope.
Note: core resources use /api instead of /apis and omit the GROUP path segment.
Examples:
/api/v1/namespaces
/api/v1/pods
/api/v1/namespaces/my-namespace/pods
/apis/apps/v1/deployments
/apis/apps/v1/namespaces/my-namespace/deployments
/apis/apps/v1/namespaces/my-namespace/deployments/my-deployment
## Request attributes used in authorization
Kubernetes reviews only the following API request attributes:

* user - The user string provided during authentication.
* group - The list of group names to which the authenticated user belongs.
* extra - A map of arbitrary string keys to string values, provided by the authentication layer.
* API - Indicates whether the request is for an API resource.
* Request path - Path to miscellaneous non-resource endpoints like /api or /healthz.
* API request verb - API verbs like get, list, create, update, patch, watch, delete, and deletecollection are used for resource requests. To determine the request verb for a resource API endpoint, see request verbs and authorization.
* HTTP request verb - Lowercased HTTP methods like get, post, put, and delete are used for non-resource requests.
* Resource - The ID or name of the resource that is being accessed (for resource requests only) -- For resource requests using get, update, patch, and delete verbs, you must provide the resource name.
* Subresource - The subresource that is being accessed (for resource requests only).
Namespace - The namespace of the object that is being accessed (for namespaced resource requests only).
* API group - The API Group being accessed (for resource requests only). An empty string designates the core API group.

## RBAC
### Referring to verb
* Non-resource requests
Requests to endpoints other than /api/v1/... or /apis/\<group>/\<version>/... are considered non-resource requests, and use the lower-cased HTTP method of the request as the verb. For example, making a GET request using HTTP to endpoints such as /api or /healthz would use <b>get</b> as the verb.
* Resource requests
To determine the request verb for a resource API endpoint, Kubernetes maps the HTTP verb used and considers whether or not the request acts on an individual resource or on a collection of resources:

|HTTP verb	|request verb|
| -------- | -------- | 
|POST	|<b>create</b>|
|GET, HEAD	|<b>get</b> (for individual resources), <b>list</b> (for collections, including full object content), <b>watch</b> (for watching an individual resource or collection of resources)|
|PUT	|<b>update</b>|
|PATCH	|<b>patch</b>|
|DELETE	|<b>delete</b> (for individual resources), <b>deletecollection</b> (for collections)|

### Referring to Resource [^1]
In the Kubernetes API, most resources are represented and accessed using a string representation of their object name, such as pods for a Pod. RBAC refers to resources using exactly the same name that appears in the URL for the relevant API endpoint. Some Kubernetes APIs involve a subresource, such as the logs for a Pod. A request for a Pod's logs looks like:
```
GET /api/v1/namespaces/{namespace}/pods/{name}/log
```
In this case, pods is the namespaced resource for Pod resources, and log is a subresource of pods. To represent this in an RBAC role, use a slash (/) to delimit the resource and subresource. To allow a subject to read pods and also access the log subresource for each of those Pods, you write:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-and-pod-logs-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]
```

A request for execute command in POD looks like:
```
POST /api/v1/namespaces/{namespace}/pods/{pod-name}/exec?container={container-name}&stdout=1&stderr=1&tty=0 HTTP/1.1  
Host: kubernetes.default.svc  
User-Agent: kubectl/vX.Y.Z (linux/amd64) kubernetes/commit-hash  
Authorization: Bearer <token>  
Content-Type: application/x-www-form-urlencoded  
  
command=<command-to-execute>&<additional-query-parameters>
```
In this case, pods is resource, exec is subresource. To allow exec in a POD, you write
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-and-pod-logs-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/exec"]
  verbs: ["get", "list", "watch", "create", "delete", "update", "patch"]
```

For more info, refer to https://kubernetes.io/docs/reference/kubernetes-api/

### Referring to subjects [^2]
A RoleBinding or ClusterRoleBinding binds a role to subjects. Subjects can be groups, users or ServiceAccounts.

A subject (user) in RoleBinding looks like:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
  namespace: development
subjects:
- kind: User
  name: dave # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

An example of subject (group) in RoleBinding is system:master group. It is a reserved group. It's bond to cluster admin role. User in system:master group is super user in K8S cluster.
```
#kubectl get clusterRoleBinding cluster-admin -owide
NAME            ROLE                      AGE   USERS   GROUPS          SERVICEACCOUNTS
cluster-admin   ClusterRole/cluster-admin 27d           system:masters
```

### Best Practice [^3]


[^1]: https://kubernetes.io/docs/reference/access-authn-authz/rbac/#referring-to-resources
[^2]: https://kubernetes.io/docs/reference/access-authn-authz/rbac/#referring-to-subjects
[^3]: https://kubernetes.io/docs/concepts/security/rbac-good-practices/
[^4]: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30/