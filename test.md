<details>
  <summary>Minimize Microservice Vulnerabilities</summary></h1>
  
## Security Contexts
```
kubectl exec ubuntu-sleeper -- whoami
```
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  securityContext:
    runAsUser: 1001    ###
  containers:
  -  image: ubuntu
     name: web
     command: ["sleep", "5000"]
     securityContext:
       runAsUser: 1002   ###
       capabilities:
         add: ["SYS_TIME"]  ###    
  -  image: ubuntu
     name: sidecar
     command: ["sleep", "5000"]
```
## Validating and Mutating Admission Controllers
webhook function
- Denies all request for pod to run as root in container if no securityContext is provided.
- If no value is set for runAsNonRoot, a default of true is applied, and the user ID defaults to 1234
- Allow to run containers as root if runAsNonRoot set explicitly to false in the securityContext
 
```
kubectl create ns webhook-demo
kubectl -n webhook-demo create secret tls webhook-server-tls \
    --cert "/root/keys/webhook-server-tls.crt" \
    --key "/root/keys/webhook-server-tls.key"
kubectl create -f /root/webhook-deployment.yaml
kubectl create -f /root/webhook-service.yaml

kubectl create -f /root/webhook-configuration.yaml    #############
```
```yaml
# /root/webhook-configuration.yaml                          ################
#apiVersion: admissionregistration.k8s.io/v1beta1
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: demo-webhook
webhooks:
  - name: webhook-server.webhook-demo.svc
    sideEffects: NoneOnDryRun   ### v1
    admissionReviewVersions: ["v1", "v1beta1"]  ### v1  
    clientConfig:
      service:
        name: webhook-server
        namespace: webhook-demo
        path: "/mutate"
      caBundle: xxxx    
    rules:     
      - operations: [ "CREATE" ]        
        apiGroups: [""]        
        apiVersions: ["v1"]        
        resources: ["pods"]
```
```
#############################
None =>   securityContext:       ### add securityContext
            runAsNonRoot: true
            runAsUser: 1234
           
securityContext:     =>     securityContext:
   runAsNonRoot: false        runAsNonRoot: false  

securityContext:
  runAsNonRoot: true
  runAsUser: 0
root@controlplane:~# kubectl apply -f /root/pod-with-conflict.yaml
Error from server: error when creating "/root/pod-with-conflict.yaml": admission webhook "webhook-server.webhook-demo.svc" denied the request: runAsNonRoot specified, but runAsUser set to 0 (the root user)
```

## Pod Security Policy
```
ps -ef | grep api | grep able  # check enable-admission-plugins or disable-admission-plugins
kube-apiserver.yaml
    - --enable-admission-plugins=NodeRestriction,PodSecurityPolicy

root@controlplane:~# cat psp.yaml 
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: example-psp
spec:
  privileged: false
  seLinux:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - configMap
  - secret
  - emptyDir
  - hostPath

kubectl apply -f /root/psp.yaml

root@controlplane:~# kubectl apply -f pod.yaml        
Error from server (Forbidden): error when creating "pod.yaml": pods "example-app" is forbidden: PodSecurityPolicy: unable to admit pod: [spec.containers[0].securityContext.privileged: Invalid value: true: Privileged containers are not allowed spec.containers[0].securityContext.capabilities.add: Invalid value: "CAP_SYS_BOOT": capability may not be added]
 
```
## OPA
## Manage Kubernetes secrets
## Using Runtime in kubernetes (gvisor, kata)
## Implement Pod to Pod encryption by mTLS
</details>
