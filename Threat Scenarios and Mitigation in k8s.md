# Scope

There are many dimensions to cover wrt securing the App and k8s cluster. The following article doesn't cover the following:
- Threat Detection
- Build Hygiene
- Image Hygiene
- SecOps

The best practice around k8s is to use _defense in depth_ (layered defense) approach and redundancy. What is simply means to have layers of protection which makes it difficult for an attacker to get further even if they get thru a single layer (to prevent breach or limit it's impact). 

## Scenario 1
Attacker is able to access k8s API server. 

### Details 
- Say due to a App side vulnerability an attacker gains access to Service Account associated with a POD and get access to the token which is stored on the POD.  
- Every SA has roles and permissions associated with it. Attacker can use the token to attack the cluster via requests to API server (in case the permissions are overly permissive and not constrained by RBAC).
- Get access to secrets to further attacks

### Mitigation 1: Use RBAC to control the permissions for SAs
- Use RBAC in k8s. Provide roles to SAs or users.
- Implementation: 
   - Create roles which are a collection of permissible actions (permissions) on the cluster.
   - RBAC setting can vary by namespaces or can apply to entire cluster.
   - tightly control the roles given to users and SAs

Permission--> Verb + Resource type (for ex _get secrets, update configmap_)   
ClusterRole--> Collection of permissions    
ClusterRoleBinding--> used to connect a user or SA to a cluster role   

So each user can have many roles associated with them. ClusterRole or ClusterRoleBinding can apply to a single namespace. 
 
### Mitigation 2: Restrict access to API server
- In GKE use _Master Authorized Network_ feature to restrict who can communicate to API server
- Firewall API server via port or regular firewall rules to certain IP addresses (so an attacker cannot even send a request or not accessible to internet)

### Mitigation 3: Network Policy based mitigation
- Restrict access to DBs to only PODs which require it
- Specify access using labels. Use [Calico](https://docs.projectcalico.org/security/calico-network-policy)/ [Weave](https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/weave-network-policy/) to implement label (POD) based rules for accepting and sending requests

## Scenario 2
Attacker is able to access cluster components and manipulate them. 

### Details
Attacker can create PODs and read secrets or modify objects. Via API server access the attacker can spin more PODs and instances, read secrets etc.

### Mitigation 1: Secure etcd
- Use Authentication & firewall to restrict access to etcd
- Encrypt data in etcd (encryption @ rest)

## Scenario 3
Attacker is able to gain access to POD Host resulting in attacks to kubelet and other containers on the host. 

### Details
This is possible in scenarios where the App is running 3rd party application or accepts data or code from the user. This allows the attacker to upload and trick the App into executing a piece of malicious code. An Attacker can also gain access thru kernel or container vulnerabilities. Attacker could read files from the FS or execute application on host outside the container. This would allow them to see and read other containers running on the host and access their secrets associated with the containers. Once access to the host is gained, attacker can read the kubelet config (authentication and certs) and attack other containers on the same host, attack the cluster and get further escalating privileges from API server. 

### Mitigation 1: Run Apps via non root (user)
- Run Apps in containers as user than root. 
- Make root FS read only. For use cases where the App needs to write some data create a temp directory or use an external volume. This prevents attackers from overwriting files from FS. So App bugs which allow file to be uploaded anywhere are mitigated. 
- Set _no_new_privs_ flag: So if the App is executing a new binary then extra privileges are not granted.

```
apiversion: v1
kind: Pod
metadata: 
   name : somename
spec: 
  securityContext:
    runAsUser: 1000
    readOnlyRootFileSystem: true
    AllowPriviligeEscalation: false
``` 
### Mitigation 2: Sandboxed PODs for less trustworthy pieces of applications (Difficult to implement generically)
- Pods are sandboxed from other pods on the same host. Sandbox allows 2 layers of isolation (Linux Kernel and Sandbox)
- Use [Kata containers](https://katacontainers.io/) or [Gvisor](https://gvisor.dev/) to have different runtimes for containers than the normal one. This allows an extra layer of isolation between container and host systems.

### Mitigation 3: Use [seccomp](https://en.wikipedia.org/wiki/Seccomp) or [AppArmor](https://wiki.ubuntu.com/AppArmor)/[SELinux](https://www.redhat.com/en/topics/linux/what-is-selinux) to create layers around the App
- App (Layer 0)
- seccomp (Layer 1)
```
apiversion: v1
kind: Pod
metadata: 
   name : somename
   annotations: 
      seccomp.security.alpha.kubernetes.io/pod: runtime/default
...
``` 
- AppArmor/ SELinux (Layer 2)

The only downside being that this needs intimate knowledge of how the App works.

### Mitigation 4: Restrict kubelet permissions
- RBAC for kubelet- special RBAC for a Node. This restricts read access of secrets on cluster for nodes  (only to the ones secrets attached to the Pods on the node)

```
--authorization-mode= RBAC,Node
--admission-control=....,NodeRestriction
```
- Rotate certs - to restrict the period during which access exists. 

(GKE takes care of these by default)

## Scenario 4
Unsecured PODs: Ensuring security standards are followed in the team while creating PODs. 

### Details
It is important to prevent creation of unsecured PODs even accidently by team members. Basically you need a policy mechanism to restrict provisioning of PODs which do not tick all the compliance boxes

### Mitigation 1: 
- Use PodSecurityPolicy. If Pods created don't match the security policy the API server will reject them. 
- Use [Open Policy Agent](https://www.openpolicyagent.org/)

## Scenario 5
Prevent attacks originating via traffic listening or intercepting activity or request forgery (MITM).

### Details
We need security at Service level to prevent attacks via _Man in the Middle mechanisms_(MITM)

### Mitigation 1: Use service mesh like Istio (with Envoy)
- Acting as a proxy between services (enforces Service level policies)
- E-to-E encryption (Encrypts traffic)
- Rolling certs (reduces the window of access in case of malicious access)
- Policy managed via central server. 

Other bits
- Use Kube-bench for ascertaining issues in cluster or actions pertaining to changes. 

[kube-bench](https://github.com/aquasecurity/kube-bench) is a Go application that checks whether Kubernetes is deployed securely by running the checks documented in the [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes/)


## Sources
- Ian Lewis, Kubernetes Forum Seoul [Kubernetes Security Best Practices](https://www.youtube.com/watch?v=wqsUfvRyYpw)
- [Sysdig Kubernetes Security guide](https://dunnhumby.sharepoint.com/:b:/r/sites/CustomerEngagementProducts/Shared%20Documents/General/k8s%20Security/kubernetes-security-guide.pdf?csf=1&web=1&e=1Qpeg9)
- Liz Rice, Container Security, O'Reilly publications
- [CKS course prep repo](https://github.com/walidshaari/Certified-Kubernetes-Security-Specialist)
