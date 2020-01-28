# Openshift: Create your own operators

An operator is a Kubernetes controller that deploys and manages a service or application in a cluster.

## Requirements
- oc cli
- go
- kubebuilder

- Model: Custom Resource Definition

```yaml
apiVersion: v1
kind: MyDatabaseCluster
metadata:
  name: my-cluster
spec:
  replicas: 3
```

## As Microservices

Kubernetes provides a rich framework for developers that enable the creation and management of microservices with relatively little code. By implementing operators as a service [1][2], the developer will benefit from the existing underlying Kubernetes conventions in addition to the highly available elements of the cluster. The resulting service will be accessible via a highly available API, backed by a replicated data store, and have built in user management and authorization. This REST API is capable of implementing all CRUD operations. Audience members will gain a new perspective of operators and their uses.

[1] https://developers.redhat.com/blog/2018/12/18/kubernetes-operators-in-depth/
[2] https://github.com/operator-framework/operator-sdk

Key takeaways:
* How using operators provides the benefits of the platform: rbac, HA, DR
* Operators are powerful utilities to help configure administer Openshift
* Using Openshift as an API provides a rich framework to the developer.

## Conclusion

Source code: https://github.com/Danil-Grigorev/failure-informer
Environment: 

oc login -u kubeadmin -p CRU6L-CbESC-Trz6d-UAzIE https://api.ci-ln-csxdsit-d5d6b.origin-ci-int-aws.dev.rhcloud.com:6443