**Question 1**: You received a list from the DevSecOps team which performed a security investigation of the cluster. The list states the following about the apiserver setup:

* Accessible through a NodePort Service

Change the apiserver setup so that:

* Only accessible through a ClusterIP Service

All the mainfest files are save to ``/etc/kubernetes/mainfest`` directory
```
vi /etc/kubernetes/mainfest/kube-apiserver.yaml
```
In there you should see a line as below
```
--kubernetes-service-node-port=31000
```

There are two way to solve this as listed below
1. Either you can change the port from ``31000`` to ``0``
```
--kubernetes-service-node-port=0 
```
OR

2. You can remove the entire line

Note: 
1. Once you do that the kube-api pod will be recreated and you may loose the connection. Wait for sometime.
2. Once the Pod is up you may still see the service exposed via NodePort. You may have to delete the service so that next time it gets created with ClusterIP.