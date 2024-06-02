# POD security
## POD Security Standard
https://kubernetes.io/docs/concepts/security/pod-security-standards/

## POD Security Admission
https://kubernetes.io/blog/2021/12/09/pod-security-admission-beta/
https://kubernetes.io/docs/concepts/security/pod-security-admission/

## Enforce POD Security Standard
https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/
This manifest defines a Namespace my-baseline-namespace that:

* Blocks any pods that don't satisfy the baseline policy requirements.
* Generates a user-facing warning and adds an audit annotation to any created pod that does not meet the restricted policy requirements.
* Pins the versions of the baseline and restricted policies to v1.30.
```
apiVersion: v1
kind: Namespace
metadata:
  name: my-baseline-namespace
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.30

    # We are setting these to our _desired_ `enforce` level.
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.30
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.30
```


## How does it work
POD Security is implemented by PodSecurity admission control module. Refer to API access control for more info about admission control of API. 
When user calls API to create a POD workload in a namespace, the POD security admission control module will check the POD specification against the namespace-wise POD security standard. This means that security sensitive fields in a pod specification will only be allowed to have specific values. If violation is found, POD creation is rejected.
好文分享：https://kubernetes.io/blog/2021/12/09/pod-security-admission-beta/

### Best Practice
https://kubernetes.io/docs/setup/best-practices/enforcing-pod-security-standards/
