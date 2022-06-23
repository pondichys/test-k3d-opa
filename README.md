This repository contains some instructions to deploy and use Open Policy Agent Gatekeeper (aka **OPA Gatekeeper**) inside a `k3d` cluster.

## What is **OPA Gatekeeper**?

**OPA Gatekeeper** enables to define and enforce policies in Kubernetes clusters.

Some sample policies can be :

- All images must be from approved registries.
- Pods must have limits.
- No workload can be deployed into the `default` namespace.

## Deploy **OPA Gatekeeper**

Create a `k3d` cluster.

```bash
k3d cluster create test-opagk
```

When the `test-opagk` cluster is ready, use `helm` to deploy **OPA Gatekeeper**.

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper/gatekeeper --name-template=gatekeeper --namespace gatekeeper-system --create-namespace
```

Check that resources are correctly deployed and running.

```bash
kubectl get all --namespace gatekeeper-system
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/gatekeeper-controller-manager-677f486574-h5w6f   1/1     Running   0          9m35s
pod/gatekeeper-controller-manager-677f486574-fnprl   1/1     Running   0          9m35s
pod/gatekeeper-controller-manager-677f486574-df2kk   1/1     Running   0          9m35s
pod/gatekeeper-audit-f7bd7bc6b-8r9fj                 1/1     Running   0          9m36s

NAME                                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/gatekeeper-webhook-service   ClusterIP   10.43.155.0   <none>        443/TCP   9m36s

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gatekeeper-controller-manager   3/3     3            3           9m36s
deployment.apps/gatekeeper-audit                1/1     1            1           9m36s

NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/gatekeeper-controller-manager-677f486574   3         3         3       9m36s
replicaset.apps/gatekeeper-audit-f7bd7bc6b                 1         1         1       9m36s
```

## Working with constraint templates and constraints

**OPA Gatekeeper** uses the [OPA Constraint Framework](https://github.com/open-policy-agent/frameworks/tree/master/constraint#readme) to describe and enforce policy. This framework is composed of 2 main components.

### Constraint template

A constraint template defines a constraint by declaring the expected input parameters and the `Rego` code that will enforce it.

Sample constraint template to check the presence of a label 

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      `rego`: |
        package k8srequiredlabels

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("you must provide labels: %v", [missing])
        }
```

You'll notice that the label key and value is not statically defined in the tempate. The template "simply" states that the label that is available as input parameter must be present.

Constraint templates are scoped at the cluster level.

### Constraint

A constraint defines that a constraint template must be enforced and how.

Sample constraint to check the presence of the label `app.kubernetes.io/name` on pods

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pods-must-have-app
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    labels: 
    - "app.kubernetes.io/name"
```

## Implementing the required label contraint

Apply the constraint template and the constraint in the Kubernetes cluster

```bash
kubectl apply --filename k8srequiredlabel_template.yml
kubectl apply --filename k8srequiredlabel_policy_pod.yml
```

Now try to apply a faulty deployment in the `default` namespace.

```bash
kubectl apply --filename bad_deployment.yml
deployment.apps/bad-deployment created
```

And thats the first surprise ... it seems that it works ! No error message ...

Let's see if it really worked ...

```bash
kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
bad-deployment   0/1     0            0           76s

# Looks like it's not working after all ... let's check
kubectl describe deployment/bad-deployment
Name:                   bad-deployment
Namespace:              default
CreationTimestamp:      Wed, 22 Jun 2022 14:42:45 +0200
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
Replicas:               1 desired | 0 updated | 0 total | 0 available | 1 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   myapp:
    Image:      nginx:1.21
    Port:       8080/TCP
    Host Port:  0/TCP
    Limits:
      cpu:        500m
      memory:     128Mi
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type             Status  Reason
  ----             ------  ------
  Progressing      True    NewReplicaSetCreated
  Available        False   MinimumReplicasUnavailable
  ReplicaFailure   True    FailedCreate
OldReplicaSets:    <none>
NewReplicaSet:     bad-deployment-8fdb5dc88 (0/1 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  2m10s  deployment-controller  Scaled up replica set bad-deployment-8fdb5dc88 to 1 

# Check the replica set
kubectl get replicaset/bad-deployment-8fdb5dc88
NAME                       DESIRED   CURRENT   READY   AGE
bad-deployment-8fdb5dc88   1         0         0       3m35s

# Deeper look into the replica set 
kubectl describe replicaset/bad-deployment-8fdb5dc88
Name:           bad-deployment-8fdb5dc88
Namespace:      default
Selector:       app=nginx,pod-template-hash=8fdb5dc88
Labels:         app=nginx
                pod-template-hash=8fdb5dc88
Annotations:    deployment.kubernetes.io/desired-replicas: 1
                deployment.kubernetes.io/max-replicas: 2
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/bad-deployment
Replicas:       0 current / 1 desired
Pods Status:    0 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=nginx
           pod-template-hash=8fdb5dc88
  Containers:
   myapp:
    Image:      nginx:1.21
    Port:       8080/TCP
    Host Port:  0/TCP
    Limits:
      cpu:        500m
      memory:     128Mi
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type             Status  Reason
  ----             ------  ------
  ReplicaFailure   True    FailedCreate
Events:
  Type     Reason        Age                   From                   Message
  ----     ------        ----                  ----                   -------
  Warning  FailedCreate  70s (x16 over 3m55s)  replicaset-controller  Error creating: admission webhook "validation.gatekeeper.sh" denied the request: [pods-must-have-app] you must provide labels: {"app.kubernetes.io/name"}
```

Here we are ... the replica set failed to create because **OPA Gatekeeper** correctly enforced the policy and found out that there was a missing required label.

Let's delete the bad-deployment and try the good one.

```bash
kubectl delete deployment/bad-deployment

kubectl apply --filename good_deployment.yml

# Let's check if it works ...
kubectl get deployment                     
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
good-deployment   1/1     1            1           29s
```

There it is !

## Cleaning up

As always, clean up your tests by deleting the whole `k3d` cluster

```bash
k3d cluster delete test-opagk
```

## Final thoughts

While **OPA Gatekeeper** is very mature product and deployed in a large number of Kubernetes clusters; I do find that there are some pain points using it. 

My main concerns are :

- Split of constraint templates and constraints. 
  There is no easy and quick way of finding what does a constraint do without having a look at the corresponding constraint template. It is a flexible way of reusing templates but makes it difficult to understand the purpose of a constraint.
- `Rego` language
  The `Rego` language is not always straigthforward and requires some time to be mastered. Now there are some nice sample constraint templates / constraints available in the [OPA library](https://github.com/open-policy-agent/gatekeeper-library) but if you need to go beyond the basic stuff, you'll have to learn serious `Rego` skills.
- No clear user notification of policy violation in some cases
  The small test we have run shows that in case of a `deployment`, the user is not immediately and clearly notified that there was a violation of a constraint. This is due to the fact that **OPA Gatekeeper** is not really Kubernetes aware.
- Reports ... or lack of
  There are no reports available with **OPA Gatekeeper** that gives an overview of the constraint's statuses.

I already use [Kyverno](https://kyverno.io) and I must say that the experience is much more pleasant. It is Kubernetes native, policies are easier to develop; you can use generating policies (policies that create another object ... can be handy) and there are some reports available (when they work though, this feature was not always stable but it exists and can only improve). Kyverno does not use `Rego` but standard `yaml` to describe policies. It makes it easier to onboard but you won't be able to define very complex policies with it ... 

Now, do you really need complex policies in your Kubernetes cluster? Only you can tell ...