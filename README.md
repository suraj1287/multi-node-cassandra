# Multi Node Cassandra on Azure using Kubernetes

## For this we will require 2 services, first one for managing internal Cassandra load balancer and other for exposing external IP
## 1.	Services

###### 1.1.	cassandra-service.yaml
```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: myapp-backend-cassandra
  name: myapp-backend-cassandra
spec:
  clusterIP: None
  ports:
    - port: 9042
  selector:
    app: myapp-backend-cassandra
```
######  1.2.	cassandra-service-ext.yaml
```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: myapp-backend-cassandra
  name: myapp-backend-cassandra-ext
spec:
  type: LoadBalancer
  ports:
    - port: 9042
  selector:
    app: myapp-backend-cassandra
```
###### 2.	persistentVolumeClaim.yaml

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myvolume-disk-claim
spec:
  storageClassName: default
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

###### 3.	cassandra-StatefullSet.yaml
```
apiVersion: "apps/v1"
kind: StatefulSet
metadata:
  name: myapp-backend-cassandra
  labels:
    app: myapp-backend-cassandra
spec:
  serviceName: myapp-backend-cassandra
  replicas: 3
  selector:
    matchLabels:
      app: myapp-backend-cassandra
  template:
    metadata:
      labels:
        app: myapp-backend-cassandra
    spec:
      containers:
        - name: myapp-backend-cassandra
          image: mycompany/myapp-cassandra
          imagePullPolicy: Always
          ports:
            - containerPort: 7000
              name: intra-node
            - containerPort: 7001
              name: tls-intra-node
            - containerPort: 7199
              name: jmx
            - containerPort: 9042
              name: cql
          env:
            - name: CASSANDRA_SEEDS
              value: myapp-backend-cassandra-0.myapp-backend-cassandra.default.svc.cluster.local
            - name: MAX_HEAP_SIZE
              value: 256M
            - name: HEAP_NEWSIZE
              value: 100M
            - name: CASSANDRA_CLUSTER_NAME
              value: "Cassandra"
            - name: CASSANDRA_DC
              value: "DC1"
            - name: CASSANDRA_RACK
              value: "Rack1"
            - name: CASSANDRA_ENDPOINT_SNITCH
              value: GossipingPropertyFileSnitch
            - name: POD_IP
              valueFrom:
                fieldRef:
                   fieldPath: status.podIP
          readinessProbe:
            exec:
              command:
              - /bin/bash
              - -c
              - /ready-probe.sh
            initialDelaySeconds: 15
            timeoutSeconds: 5        
          volumeMounts:
          - mountPath: /var/lib/cassandra/data
            name: myvolume-disk-claim
  volumeClaimTemplates:
  - metadata:
      name: myvolume-disk-claim
    spec:
      storageClassName: default
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
```

## Steps to deploy on Azure Kubernetes cluster

###### 1.	az login --service-principal -u  xxxx00000-0000-000-xxxx-000000000000 --password  pass@123 --tenant xxxx00000-0000-000-xxxx-000xxxxxxxx
```
[
  {
    "cloudName": "AzureCloud",
    "id": "xxxx00000-0000-000-xxxx-000xxxxxxxx",
    "isDefault": true,
    "name": "Azure",
    "state": "Enabled",
    "tenantId": "xxxx00000-0000-000-xxxx-000xxxxxxxx",
    "user": {
      "name": "xxxx00000-0000-000-xxxx-000000000000",
      "type": "servicePrincipal"
    }
  }
]
```
###### 2.	az group create -l eastus -n myapp-dl
```
{
  "id": "/subscriptions/xxxx00000-0000-000-xxxx-000xxxxxxxx/resourceGroups/myapp-dl",
  "location": "eastus",
  "managedBy": null,
  "name": "myapp-dl",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null
}
```
###### 3.	sudo az aks create --resource-group myapp-dl --name myapp-cass-aks --node-count 3 --nodepool-name myappcassaks --node-vm-size Standard_A4_v2 --enable-addons monitoring --service-principal xxxx00000-0000-000-xxxx-000000000000 --client-secret pass@123
```
{
  "aadProfile": null,
  "addonProfiles": {
    "omsagent": {
      "config": {
        "logAnalyticsWorkspaceResourceID": "/subscriptions/xxxx00000-0000-000-xxxx-000xxxxxxxx/resourcegroups/defaultresourcegroup-eus/providers/microsoft.operationalinsights/workspaces/defaultworkspace-xxxx00000-0000-000-xxxx-000xxxxxxxx-eus"
      },
      "enabled": true
    }
  },
  "agentPoolProfiles": [
    {
      "count": 3,
      "maxPods": 110,
      "name": "myappcassaks",
      "osDiskSizeGb": 100,
      "osType": "Linux",
      "storageProfile": "ManagedDisks",
      "vmSize": "Standard_A4_v2",
      "vnetSubnetId": null
    }
  ],
  "dnsPrefix": "myapp-cass-myapp-dl-b6bee4",
  "enableRbac": true,
  "fqdn": "myapp-cass-myapp-dl-b6bee4-d806470c.hcp.eastus.azmk8s.io",
  "id": "/subscriptions/xxxx00000-0000-000-xxxx-000xxxxxxxx/resourcegroups/myapp-dl/providers/Microsoft.ContainerService/managedClusters/myapp-cass-aks",
  "kubernetesVersion": "1.11.8",
  "linuxProfile": {
    "adminUsername": "azureuser",
    "ssh": {
      "publicKeys": [
        {
          "keyData": "ssh-rsa AAAAB3NzaC1yc2EAAajbdjbdjQDxzNPbRtO1ZbZ/LVEuFvNvEsSxtvL7R0r450yY9ieAEsnm5KLPAJLnFqmZ4Wq8WAu+DkjwbfjbwfSCabO7dtxuSSJeN98aDyJtAsM2BbSgCQNb0ENRXL6giBULPlnH6A6Gm1Sw02H+R5zobA7S+YVRnd2fW3X9hAufitSu+pTkVbLsOBvYzd+HhveYo4XP6kO2NuMgFqXCqu1GQbac8ZZvunGQKNKRWzj7VI4HnqyNq+CKqr7HUhEkWDFEAiJIEVJ4yHgq/BTQ9nMmHdiK77EP0kRv7Agcyi4yK3WgFA5j2CmNmvyDt4OFvrQyW/LSHyh/x1FkNbES9zeY6p730ir"
        }
      ]
    }
  },
  "location": "eastus",
  "name": "myapp-cass-aks",
  "networkProfile": {
    "dnsServiceIp": "10.0.0.10",
    "dockerBridgeCidr": "172.17.0.1/16",
    "networkPlugin": "kubenet",
    "networkPolicy": null,
    "podCidr": "10.244.0.0/16",
    "serviceCidr": "10.0.0.0/16"
  },
  "nodeResourceGroup": "MC_myapp-dl_myapp-cass-aks_eastus",
  "provisioningState": "Succeeded",
  "resourceGroup": "myapp-dl",
  "servicePrincipalProfile": {
    "clientId": "xxxx00000-0000-000-xxxx-000000000000",
    "secret": null
  },
  "tags": null,
  "type": "Microsoft.ContainerService/ManagedClusters"
}
```
###### 4.	sudo az aks get-credentials --resource-group myapp-dl --name myapp-cass-aks --overwrite-existing
```
Merged "myapp-cass-aks" as current context in /home/dspg/.kube/config
```
###### 5.	sudo kubectl get storageclasses
```
NAME                PROVISIONER                AGE
default (default)   kubernetes.io/azure-disk   11m
managed-premium     kubernetes.io/azure-disk   11m
```
###### 6.	sudo kubectl apply -f persistentVolumeClaimDisk.yaml
```
persistentvolumeclaim/myvolume-disk-claim created
```
###### 7.	sudo kubectl get pvc
```
NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myvolume-disk-claim   Bound    pvc-dhkll4f5b-5074-11e9-afef-2a1djfbx80e0a   10Gi       RWO            default        25s
```


###### 8.	sudo kubectl create -f cassandra-service.yaml
```
service/myapp-backend-cassandra created
```
###### 9.	sudo kubectl create -f cassandra-service-ext.yaml
```
service/myapp-backend-cassandra-ext created
```
###### 10.	sudo kubectl create -f cassandra-statefulset.yaml
```
statefulset.apps/myapp-backend-cassandra created
```

###### 11.	sudo kubectl get pods -o wide
```
NAME                        READY   STATUS    RESTARTS   AGE
myapp-backend-cassandra-0   1/1     Running   0          1h
myapp-backend-cassandra-1   1/1     Running   0          1h
myapp-backend-cassandra-2   1/1     Running   0          1h
```

###### 12.	kubectl get svc
```
NAME                          TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
kubernetes                    ClusterIP      10.0.0.1      <none>        443/TCP          18m
myapp-backend-cassandra       ClusterIP      None          <none>        9042/TCP         2m
myapp-backend-cassandra-ext   LoadBalancer   10.0.78.189   19.76.35.58   9042:32133/TCP   2m
```
###### 13.	cqlsh 56.44.55.22
```
dspg@mycompanyds28:~/cassandra-spark-testing$ cqlsh 56.44.55.22
Connected to Cassandra at 56.44.55.22:9042.
[cqlsh 5.0.1 | Cassandra 3.11.2 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh> exit 
```
###### 14.	kubectl exec -it myapp-backend-cassandra-0 – nodetool status
```
Datacenter: DC1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address      Load       Tokens       Owns (effective)  Host ID                               Rack
UN  10.244.2.4   137.44 MiB  32           100.0%            1c14f773-7e64-45ae-afc2-515af63a3f63  Rack1
UN  10.244.1.3   137.32 MiB  32           100.0%            6224125f-977d-4d53-90d3-8648302dbf62  Rack1
UN  10.244.0.10  137.28 MiB  32           100.0%            db8c59fe-554b-428c-9bbf-9778289a7b82  Rack1
```
###### 15.	Submit spark job giving Cassandra cluster ip along with keyspace name and sqlserver source ip
```
root@spark-master:/spark/bin# ./spark-submit --class com.mycompany.myapp.stream.consumer.TimelogsRealTimeWithoutKafka --master spark://192.168.100.104:6066 --deploy-mode cluster   --executor-memory 2G --executor-cores 2 --supervise  --verbose /spark/jobs/myapp.jar spark://192.168.100.104:7077 56.44.55.22 myapp 192.168.101.138
```
###### 16.	cqlsh> select * from myapp.timelogs limit 1;
```
 block_id | taskid    | userid   | timelogdate | activitycode | applicationcode | block_type | empid | enddate    | hours | isunplanned | iterationcode | modifieddate                    | modulecode | projectcode | projectid | remaininghours | requestcode | startdate  | subprojectcode | syncdate                        | task_name | taskuid
----------+-----------+----------+-------------+--------------+-----------------+------------+-------+------------+-------+-------------+---------------+---------------------------------+------------+-------------+-----------+----------------+-------------+------------+----------------+---------------------------------+-----------+---------
70033533 | 101548688 | 71241548 |  2015-04-15 |        PIARV |            null |      Enh_f |  null | 2015-04-16 |  0.40 |           N |               | 2016-04-27 21:17:49.980000+0000 |       null |     IS1CPMS |  70004464 |          -1.00 |     MinE479 | 2015-04-15 |  IS1CPMSMin479 | 1970-01-01 00:00:00.000000+0000 | Review IA |   32523
(1 rows)
```
###### 17.	kubectl exec -it myapp-backend-cassandra-0 -- nodetool getendpoints myapp timelogs 70033533
```
10.244.1.3
10.244.2.4
10.244.0.10
```

