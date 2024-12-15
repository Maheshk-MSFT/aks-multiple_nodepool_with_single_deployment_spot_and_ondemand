# multiple_nodepool_with_single_deployment yaml
Have spot pool and on-demand mix, but start with spot1, spot2 pools and then land finally on-demand

## 1. Make sure we have right labels set for the nodepool and tainted
<img width="892" alt="1" src="https://github.com/user-attachments/assets/327a527a-a221-43ba-a08e-efef587f0a6d" />

<img width="877" alt="2" src="https://github.com/user-attachments/assets/70ebc6f4-eb85-4e5e-89b1-4219f84d16df" />

<img width="843" alt="3" src="https://github.com/user-attachments/assets/700f5b59-2587-4510-9e42-acc652d04220" />

<img width="886" alt="4" src="https://github.com/user-attachments/assets/a0b3f01b-f10a-4534-b745-fff9b3992410" />

<img width="884" alt="5" src="https://github.com/user-attachments/assets/9d135717-5bc0-4f31-8d6f-727faa5f8732" />

<img width="877" alt="6" src="https://github.com/user-attachments/assets/987ecd0e-1aec-4148-b0ad-1eb1becad9df" />

## 2. update the cluster-autoscaler-profile expander from Random to Priority - this is key thing to deploy across the nodepools
```
az aks nodepool update --resource-group <rg> --cluster-name <clustername> --cluster-autoscaler-profile expander=priority
```

## 3. Create this config map (OPTIONAL - no effect even if we delete this cm. We used to create this CM in the past but looks like this step is optional. if it works without this, then skip this step)

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-priority-expander
  namespace: kube-system
data:
  priorities: |-
    1:
      - ".*"
    10:
      - ".*odpool.*"
    50:
      - ".spotpl2.*"
    70:
      - ".*spotpl.*"
```
## 4. Follow this yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-nodepool-app
  namespace: platform
  labels:
    app: multi-nodepool-app
spec:
  replicas: 50
  selector:
    matchLabels:
      app: multi-nodepool-app
  template:
    metadata:
      labels:
        app: multi-nodepool-app
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: nodetype
                    operator: In
                    values:
                      - spotpl1
                      - spotpl2
                      - odpool 
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: workload
                    operator: In
                    values:
                      - testspot1
            - weight: 50
              preference:
                matchExpressions:
                  - key: workload
                    operator: In
                    values:
                      - testspot2
            - weight: 10
              preference:
                matchExpressions:
                  - key: workload
                    operator: In
                    values:
                      - testod
      containers:
        - name: app-container
          image: nginx:latest
          resources:
            requests:
              memory: "500Mi"
              cpu: "0.5"
            limits:
              memory: "1Gi"
              cpu: "1"
          ports:
            - containerPort: 80
      tolerations:
        - key: "workload"
          operator: "Equal"
          value: "testspot1"
          effect: "NoSchedule"
        - key: "workload"
          operator: "Equal"
          value: "testspot2"
          effect: "NoSchedule"
        - key: "kubernetes.azure.com/scalesetpriority"
          operator: "Equal"
          value: "spot"
          effect: "NoSchedule"
        - key: "workload"
          operator: "Equal"
          value: "testod"
          effect: "NoSchedule"
```

## useful links
-------------
https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#what-are-expanders
<br>
https://www.youtube.com/watch?v=mo2UrkjA7FE
<br>
https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/expander/priority/readme.md
<br>
https://learn.microsoft.com/en-us/azure/aks/cluster-autoscaler-overview#best-practices-and-considerations [Read the 3 bullet point]
<img width="466" alt="image" src="https://github.com/user-attachments/assets/c7316e7e-e432-497f-a259-ed9ab4a4edf1" />
Using Spot Node with Karpenter (NAP) mixing sku's -> https://techcommunity.microsoft.com/blog/appsonazureblog/karpenter-run-your-workloads-upto-80-off-using-spot-with-aks/4148840

## useful commands
---------------
az aks show -n <clustername> -g <res-grp> --query autoScalerProfile.expander
<br>
az aks nodepool show --resource-group <res_grp_name> --cluster-name <cluster_name> --name <nodepool_name>
<br>
check if the scaleset priority is set to Spot -> "scaleSetPriority": "Spot",
<br>
