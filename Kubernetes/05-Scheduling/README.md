kubectl taint nodes rm-cluster-2-worker dedicated=database:NoSchedule   #add taint
kubectl taint nodes rm-cluster-2-worker dedicated=database:NoSchedule-  #remove taint

kubectl get pods -l app=prd         # select pods matching with prd label
kubectl label pod nginx environment=prd   #add label
kubectl label pod nginx environment-        #delete label
kubectl label pod nginx environment=dev  #overwrite label
kubectl get pods --show-label    # show pods with labels
kubectl get pods -l 'env in (dev,production)' # select pods multit label condition

#only matching taint node matching operator can schedule pod
tolerations:
  - key: dedicated
    operator: Equal
    value: database
    effect: NoSchedule

#existing any taint node can tolerate
tolerations:
  - key: dedicated
    operator: Exists
    effect: NoSchedule


Affinity : Mera Pod kis Node par ya kis Pod ke paas run hona chahiye
│
├── Node Affinity
│     └── Particular Node choose
│
├── Pod Affinity
│     └── Matching Pod ke paas
│
└── Pod Anti-Affinity
      └── Matching Pod se door

Node label:
Add label
kubectl label node rm-cluster-2-worker disktype=ssd
kubectl label node rm-cluster-2-worker2 disktype=hdd

Remove label
kubectl label node rm-cluster-2-worker disktype-
kubectl get nodes --show-labels