# multiple_nodepool_with_single_deployment
Have spot nodes and ondemand but start with spot1, spot2 pools then land finally on-demand
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
