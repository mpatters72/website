---
reviewers:
- liggitt
- mikedanese
- munnerz
title: Certificate Signing Requests
content_type: concept
weight: 20
---

<!-- overview -->

{{< feature-state for_k8s_version="v1.18" state="beta" >}}

The Certificates API enables automation of
[X.509](https://www.itu.int/rec/T-REC-X.509) credential provisioning by providing
a programmatic interface for clients of the Kubernetes API to request and obtain
X.509 {{< glossary_tooltip term_id="certificate" text="certificates" >}} from a Certificate Authority (CA).

A CertificateSigningRequest (CSR) resource is used to request that a certificate be signed
by a denoted signer, after which the request may be approved or denied before
finally being signed.



<!-- body -->
## Request signing process

The _CertificateSigningRequest_ resource type allows a client to ask for an X.509 certificate
be issued, based on a signing request.
The CertificateSigningRequest object includes a PEM-encoded PKCS#10 signing request in
the `spec.request` field. The CertificateSigningRequest denotes the _signer_ (the
recipient that the request is being made to) using the `spec.signerName` field.

Once created, a CertificateSigningRequest must be approved before it can be signed.
Depending on the signer selected, a CertificateSigningRequest may be automatically approved
by a {{< glossary_tooltip text="controller" term_id="controller" >}}.
Otherwise, a CertificateSigningRequest must be manually approved either via the REST API (or client-go)
or by running `kubectl certificate approve`. Likewise, a CertificateSigningRequest may also be denied,
which tells the configured signer that it must not sign the request.

For certificates that have been approved, the next step is signing. The relevant signing controller
first validates that the signing conditions are met and then creates a certificate.
The signing controller then updates the CertificateSigningRequest, storing the new certificate into
the `status.certificate` field of the existing CertificateSigningRequest object. The
`status.certificate` field is either empty or contains a X.509 certificate, encoded in PEM format.
The CertificateSigningRequest `status.certificate` field is empty until the signer does this.

Once the `status.certificate` field has been populated, the request has been completed and clients can now
fetch the signed certificate PEM data from the CertificateSigningRequest resource.
The signers can instead deny certificate signing if the approval conditions are not met.

In order to reduce the number of old CertificateSigningRequest resources left in a cluster, a garbage collection
controller runs periodically. The garbage collection removes CertificateSigningRequests that have not changed
state for some duration:

* Approved requests: automatically deleted after 1 hour
* Denied requests: automatically deleted after 1 hour
* Pending requests: automatically deleted after 1 hour

## Signers

All signers should provide information about how they work so that clients can predict what will happen to their CSRs.
This includes:

1. **Trust distribution**: how trust (CA bundles) are distributed.
1. **Permitted subjects**: any restrictions on and behavior when a disallowed subject is requested.
1. **Permitted x509 extensions**: including IP subjectAltNames, DNS subjectAltNames, Email subjectAltNames, URI subjectAltNames etc, and behavior when a disallowed extension is requested.
1. **Permitted key usages / extended key usages**: any restrictions on and behavior when usages different than the signer-determined usages are specified in the CSR.
1. **Expiration/certificate lifetime**: whether it is fixed by the signer, configurable by the admin, determined by the CSR object etc and the behavior when an expiration is different than the signer-determined expiration that is specified in the CSR.
1. **CA bit allowed/disallowed**: and behavior if a CSR contains a request a for a CA certificate when the signer does not permit it.

Commonly, the `status.certificate` field contains a single PEM-encoded X.509 certificate once the CSR is approved and the certificate is issued. Some signers store multiple certificates into the `status.certificate` field. In that case, the documentation for the signer should specify the meaning of additional certificates; for example, this might be the certificate plus intermediates to be presented during TLS handshakes.

### Kubernetes signers

Kubernetes provides built-in signers that each have a well-known `signerName`:

1. `kubernetes.io/kube-apiserver-client`: signs certificates that will be honored as client-certs by the kube-apiserver.
  Never auto-approved by {{< glossary_tooltip term_id="kube-controller-manager" >}}.
    1. Trust distribution: signed certificates must be honored as client-certificates by the kube-apiserver. The CA bundle is not distributed by any other means.
    1. Permitted subjects - no subject restrictions, but approvers and signers may choose not to approve or sign. Certain subjects like cluster-admin level users or groups vary between distributions and installations, but deserve additional scrutiny before approval and signing. The `CertificateSubjectRestriction` admission plugin is available and enabled by default to restrict `system:masters`, but it is often not the only cluster-admin subject in a cluster.
    1. Permitted x509 extensions - honors subjectAltName and key usage extensions and discards other extensions.
    1. Permitted key usages - must include []string{"client auth"}. Must not include key usages beyond []string{"digital signature", "key encipherment", "client auth"}
    1. Expiration/certificate lifetime - minimum of CSR signer or request. The signer is responsible for checking that the certificate lifetime is valid and permissible.
    1. CA bit allowed/disallowed - not allowed.

1. `kubernetes.io/kube-apiserver-client-kubelet`: signs client certificates that will be honored as client-certs by the
  kube-apiserver.
  May be auto-approved by {{< glossary_tooltip term_id="kube-controller-manager" >}}.
    1. Trust distribution: signed certificates must be honored as client-certificates by the kube-apiserver. The CA bundle
       is not distributed by any other means.
    1. Permitted subjects - organizations are exactly `[]string{"system:nodes"}`, common name starts with `"system:node:"`
    1. Permitted x509 extensions - honors key usage extensions, forbids subjectAltName extensions and drops other extensions.
    1. Permitted key usages - exactly `[]string{"key encipherment", "digital signature", "client auth"}`
    1. Expiration/certificate lifetime - minimum of CSR signer or request. The signer is responsible for checking that the certificate lifetime is valid and permissible.
    1. CA bit allowed/disallowed - not allowed.

1. `kubernetes.io/kubelet-serving`: signs serving certificates that are honored as a valid kubelet serving certificate
  by the kube-apiserver, but has no other guarantees.
  Never auto-approved by {{< glossary_tooltip term_id="kube-controller-manager" >}}.
    1. Trust distribution: signed certificates must be honored by the kube-apiserver as valid to terminate connections to a kubelet. The CA bundle is not distributed by any other means.
    1. Permitted subjects - organizations are exactly `[]string{"system:nodes"}`, common name starts with `"system:node:"`
    1. Permitted x509 extensions - honors key usage and DNSName/IPAddress subjectAltName extensions, forbids EmailAddress and URI subjectAltName extensions, drops other extensions. At least one DNS or IP subjectAltName must be present.
    1. Permitted key usages - exactly `[]string{"key encipherment", "digital signature", "server auth"}`
    1. Expiration/certificate lifetime - minimum of CSR signer or request.
    1. CA bit allowed/disallowed - not allowed.

1. `kubernetes.io/legacy-unknown`:  has no guarantees for trust at all. Some distributions may honor these as client
  certs, but that behavior is non-standard Kubernetes behavior.
  Never auto-approved by {{< glossary_tooltip term_id="kube-controller-manager" >}}.
    1. Trust distribution: None.  There is no standard trust or distribution for this signer in a Kubernetes cluster.
    1. Permitted subjects - any
    1. Permitted x509 extensions - honors subjectAltName and key usage extensions and discards other extensions.
    1. Permitted key usages - any
    1. Expiration/certificate lifetime - minimum of CSR signer or request. The signer is responsible for checking that the certificate lifetime is valid and permissible.
    1. CA bit allowed/disallowed - not allowed.

{{< note >}}
Failures for all of these are only reported in kube-controller-manager logs.
{{< /note >}}

Distribution of trust happens out of band for these signers.  Any trust outside of those described above are strictly
coincidental. For instance, some distributions may honor `kubernetes.io/legacy-unknown` as client certificates for the
kube-apiserver, but this is not a standard.
None of these usages are related to ServiceAccount token secrets `.data[ca.crt]` in any way.  That CA bundle is only
guaranteed to verify a connection to the kube-apiserver using the default service (`kubernetes.default.svc`).

## Authorization

To allow creating a CertificateSigningRequest and retrieving any CertificateSigningRequest:

* Verbs: `create`, `get`, `list`, `watch`, group: `certificates.k8s.io`, resource: `certificatesigningrequests`

For example:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: csr-creator
rules:
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests
  verbs:
  - create
  - get
  - list
  - watch
```

To allow approving a CertificateSigningRequest:

* Verbs: `get`, `list`, `watch`, group: `certificates.k8s.io`, resource: `certificatesigningrequests`
* Verbs: `update`, group: `certificates.k8s.io`, resource: `certificatesigningrequests/approval`
* Verbs: `approve`, group: `certificates.k8s.io`, resource: `signers`, resourceName: `<signerNameDomain>/<signerNamePath>` or `<signerNameDomain>/*`

For example:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: csr-approver
rules:
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests/approval
  verbs:
  - update
- apiGroups:
  - certificates.k8s.io
  resources:
  - signers
  resourceNames:
  - example.com/my-signer-name # example.com/* can be used to authorize for all signers in the 'example.com' domain
  verbs:
  - approve
```

To allow signing a CertificateSigningRequest:

* Verbs: `get`, `list`, `watch`, group: `certificates.k8s.io`, resource: `certificatesigningrequests`
* Verbs: `update`, group: `certificates.k8s.io`, resource: `certificatesigningrequests/status`
* Verbs: `sign`, group: `certificates.k8s.io`, resource: `signers`, resourceName: `<signerNameDomain>/<signerNamePath>` or `<signerNameDomain>/*`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: csr-signer
rules:
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests/status
  verbs:
  - update
- apiGroups:
  - certificates.k8s.io
  resources:
  - signers
  resourceName:
  - example.com/my-signer-name # example.com/* can be used to authorize for all signers in the 'example.com' domain
  verbs:
  - sign
```

## Normal User

There are a few steps are required in order to get normal user to be able to authenticate and invoke API. First, this user must have certificate issued by the Kubernetes Cluster, and then present that Certificate into the API call as the Certificate Header, or through the kubectl.

### Create Private Key

The following scripts show how to generate PKI private key and CSR. It is important to set CN and O attribute of the CSR. CN is the name of the user and O is the group that this user will belong to. You can refer to [RBAC](/docs/reference/access-authn-authz/rbac/) for standard groups.

```
openssl genrsa -out john.key 2048
openssl req -new -key john.key -out john.csr
```

### Create Certificate Request Kubernetes Object

Create a CertificateSigningRequest and submit it to a Kubernetes Cluster via kubectl. Below is a script to generate the CertificateSigningRequest.

```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: john
spec:
  groups:
  - system:authenticated
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZVzVuWld4aE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQTByczhJTHRHdTYxakx2dHhWTTJSVlRWMDNHWlJTWWw0dWluVWo4RElaWjBOCnR2MUZtRVFSd3VoaUZsOFEzcWl0Qm0wMUFSMkNJVXBGd2ZzSjZ4MXF3ckJzVkhZbGlBNVhwRVpZM3ExcGswSDQKM3Z3aGJlK1o2MVNrVHF5SVBYUUwrTWM5T1Nsbm0xb0R2N0NtSkZNMUlMRVI3QTVGZnZKOEdFRjJ6dHBoaUlFMwpub1dtdHNZb3JuT2wzc2lHQ2ZGZzR4Zmd4eW8ybmlneFNVekl1bXNnVm9PM2ttT0x1RVF6cXpkakJ3TFJXbWlECklmMXBMWnoyalVnald4UkhCM1gyWnVVV1d1T09PZnpXM01LaE8ybHEvZi9DdS8wYk83c0x0MCt3U2ZMSU91TFcKcW90blZtRmxMMytqTy82WDNDKzBERHk5aUtwbXJjVDBnWGZLemE1dHJRSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBR05WdmVIOGR4ZzNvK21VeVRkbmFjVmQ1N24zSkExdnZEU1JWREkyQTZ1eXN3ZFp1L1BVCkkwZXpZWFV0RVNnSk1IRmQycVVNMjNuNVJsSXJ3R0xuUXFISUh5VStWWHhsdnZsRnpNOVpEWllSTmU3QlJvYXgKQVlEdUI5STZXT3FYbkFvczFqRmxNUG5NbFpqdU5kSGxpT1BjTU1oNndLaTZzZFhpVStHYTJ2RUVLY01jSVUyRgpvU2djUWdMYTk0aEpacGk3ZnNMdm1OQUxoT045UHdNMGM1dVJVejV4T0dGMUtCbWRSeEgvbUNOS2JKYjFRQm1HCkkwYitEUEdaTktXTU0xMzhIQXdoV0tkNjVoVHdYOWl4V3ZHMkh4TG1WQzg0L1BHT0tWQW9FNkpsYWFHdTlQVmkKdjlOSjVaZlZrcXdCd0hKbzZXdk9xVlA3SVFjZmg3d0drWm89Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  usages:
  - client auth
EOF
```

Some points to note:

- usage has to be 'client auth'
- request is the base64 encoded value of the CSR file content. You can use this command to get that ```cat john.csr | base64 | tr -d "\n"```

### Approve Certificate Request

Use kubeadmin to create a CSR and approve it.

Get the list of CSRs
```
kubectl get csr
```

Approve the CSR
```
kubectl certificate approve john
```

### Get the Certificate

Retrieve the Certificate from the CSR.

```
kubectl get csr/john -o yaml
```

The Certificate value is in Base64-encoded format under status.certificate.

Export the issued certificate from the CertificateSigningRequest.

```
kubectl get csr john -o jsonpath='{.status.certificate}'| base64 -d > /home/vagrant/work/john.crt
```

### Create Role and Role Binding

You get the Certificate already. Now it is time to define the Role and Role Binding for this user to access Kubernetes Cluster resources.

This is a sample script to create role for this new user
```
kubectl create role developer --verb=create --verb=get --verb=list --verb=update --verb=delete --resource=pods
```

This is a sample script to create role binding for this new user
```
kubectl create rolebinding developer-binding-john --role=developer --user=john
```

### Add to KubeConfig

The last step is to add this user into the KubeConfig. We assume the key and crt files are located here "/home/vagrant/work/".

First, we need to add new credentials
```
kubectl config set-credentials john --client-key=/home/vagrant/work/john.key --client-certificate=/home/vagrant/work/john.crt --embed-certs=true

```

Then, we need to add the context
```
kubectl config set-context john --cluster=kubernetes --user=john
```

To test it, change kubecontext to john
```
kubectl config use-context john
```

## Approval & rejection

### Control plane automated approval {#approval-rejection-control-plane}

The kube-controller-manager ships with a built-in approver for certificates with
a signerName of `kubernetes.io/kube-apiserver-client-kubelet` that delegates various
permissions on CSRs for node credentials to authorization.
The kube-controller-manager POSTs SubjectAccessReview resources to the API server
in order to check authorization for certificate approval.

### Approval & rejection using `kubectl` {#approval-rejection-kubectl}

A Kubernetes administrator (with appropriate permissions) can manually approve
(or deny) CertificateSigningRequests by using the `kubectl certificate
approve` and `kubectl certificate deny` commands.

To approve a CSR with kubectl:

```bash
kubectl certificate approve <certificate-signing-request-name>
```

Likewise, to deny a CSR:

```bash
kubectl certificate deny <certificate-signing-request-name>
```

### Approval & rejection using the Kubernetes API {#approval-rejection-api-client}

Users of the REST API can approve CSRs by submitting an UPDATE request to the `approval`
subresource of the CSR to be approved. For example, you could write an
{{< glossary_tooltip term_id="operator-pattern" text="operator" >}} that watches for a particular
kind of CSR and then sends an UPDATE to approve them.

When you make an approval or rejection request, set either the `Approved` or `Denied`
status condition based on the state you determine:

For `Approved` CSRs:

```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
...
status:
  conditions:
  - lastUpdateTime: "2020-02-08T11:37:35Z"
    message: Approved by my custom approver controller
    reason: ApprovedByMyPolicy # You can set this to any string
    type: Approved
```

For `Denied` CSRs:

```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
...
status:
  conditions:
  - lastUpdateTime: "2020-02-08T11:37:35Z"
    message: Denied by my custom approver controller
    reason: DeniedByMyPolicy # You can set this to any string
    type: Denied
```

It's usual to set `status.conditions.reason` to a machine-friendly reason
code using TitleCase; this is a convention but you can set it to anything
you like. If you want to add a note just for human consumption, use the
`status.conditions.message` field.

## Signing

### Control plane signer {#signer-control-plane}

The Kubernetes control plane implements each of the [Kubernetes signers](/docs/reference/access-authn-authz/certificate-signing-requests/#kubernetes-signers),
as part of the kube-controller-manager.

{{< note >}}
Prior to Kubernetes v1.18, the kube-controller-manager would sign any CSRs that
were marked as approved.
{{< /note >}}

### API-based signers {#signer-api}

Users of the REST API can sign CSRs by submitting an UPDATE request to the `status`
subresource of the CSR to be signed.

As part of this request, the `status.certificate` field should be set to contain the
signed certificate.



## {{% heading "whatsnext" %}}


* Read [Manage TLS Certificates in a Cluster](/docs/tasks/tls/managing-tls-in-a-cluster/)
* View the source code for the kube-controller-manager built in [signer](https://github.com/kubernetes/kubernetes/blob/32ec6c212ec9415f604ffc1f4c1f29b782968ff1/pkg/controller/certificates/signer/cfssl_signer.go)
* View the source code for the kube-controller-manager built in [approver](https://github.com/kubernetes/kubernetes/blob/32ec6c212ec9415f604ffc1f4c1f29b782968ff1/pkg/controller/certificates/approver/sarapprove.go)
* For details of X.509 itself, refer to [RFC 5280](https://tools.ietf.org/html/rfc5280#section-3.1) section 3.1
* For information on the syntax of PKCS#10 certificate signing requests, refer to [RFC 2986](https://tools.ietf.org/html/rfc2986)
