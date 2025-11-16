# 자습서: AKS(Azure Kubernetes Service)에서 애플리케이션 크기 조정

> 원문: https://learn.microsoft.com/ko-kr/azure/aks/tutorial-kubernetes-scale?tabs=azure-cli  

이 자습서는 이전 단계에서 만든 AKS 클러스터와 Azure Store Front 예제 애플리케이션을 기반으로,  
클러스터와 애플리케이션을 **확장(Scale)** 하는 여러 방법을 실습하는 단계입니다.

---

## 이 문서의 내용

이 자습서에서는 다음 내용을 다룹니다.

- Kubernetes 노드 수를 늘리거나 줄이는 방법
- 애플리케이션을 실행하는 Kubernetes Pod 수를 **수동으로** 조정하는 방법
- 프런트엔드 Pod에 대해 **수평 Pod 자동 크기 조정(HPA)** 를 구성하는 방법

---

## 수동으로 Pod 크기 조정

1. `kubectl get` 명령을 사용하여 클러스터에서 노드를 봅니다.

```bash
kubectl get pods
```
다음 예제 출력은 Azure Store Front 앱을 실행하는 Pod를 보여 줍니다.
```출력
NAME                               READY     STATUS     RESTARTS   AGE
order-service-848767080-tf34m      1/1       Running    0          31m
product-service-4019737227-2q2qz   1/1       Running    0          31m
store-front-2606967446-2q2qz       1/1       Running    0          31m
```

2. 명령을 사용하여 `kubectl scale` 배포의 Pod 수를 수동으로 변경합니다
```콘솔
kubectl scale --replicas=5 deployment.apps/store-front
```
다음 예제 출력은 Azure Store Front 앱을 실행하는 추가 Pod를 보여줍니다.
```출력
NAME                              READY     STATUS    RESTARTS   AGE
store-front-3309479140-2hfh0      1/1       Running   0          3m
store-front-3309479140-bzt05      1/1       Running   0          3m
store-front-3309479140-fvcvm      1/1       Running   0          3m
store-front-3309479140-hrbf2      1/1       Running   0          15m
store-front-3309479140-qphz8      1/1       Running   0          3m
```
---

## Pod 자동 크기 조정

수평방향 Pod 자동크기조정을 사용하려면 모든 컨테이너에 정의된 CPU Request 및 Limit이 있어야 하며 Pod에는 지정된 요청이 있어야 합니다.
`aks-store-quickstart`배포에서 프론트엔드 컨테이너는 1m CPU를 요청하며 제한은 1000m CPU 입니다.

다음 압축된 YAML 예제에 나와 있는 것처럼 이러한 리소스 요청과 제한은 각 컨테이너에 대해 정의됩니다.

```YAML
...
  containers:
  - name: store-front
    image: ghcr.io/azure-samples/aks-store-demo/store-front:latest
    ports:
    - containerPort: 8080
      name: store-front
...
    resources:
      requests:
        cpu: 1m
...
      limits:
        cpu: 1000m
...
```
---

## 매니페스트 파일을 사용하여 Pod 자동 크기 조정

1.다음 예제의 압축된 매니페스트 파일 aks-store-quickstart-hpa.yaml에 표시된 것처럼 자동 크기 조정기 동작 및 리소스 제한을 정의하는 매니페스트 파일을 만듭니다.

```YAML
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: store-front-hpa
spec:
  maxReplicas: 10 # define max replica count
  minReplicas: 3  # define min replica count
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: store-front
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

2. `kubectl apply` 명령을 사용하여 자동 크기 조정기 매니페스트 파일을 적용합니다.

```콘솔
kubectl apply -f aks-store-quickstart-hpa.yaml
```

3. `kubectl get hpa` 명령을 사용하여 자동 크기 조정기의 상태를 확인합니다.

```콘솔
kubectl get hpa
```

Azure Store Front 앱에 최소 부하를 적용한 상태로 몇 분이 지나면 Pod 복제본 수가 3개로 줄어듭니다. 명령을 다시 사용하여 kubectl get pods 불필요한 Pod가 제거되는 것을 볼 수 있습니다.

---

## 수동으로 AKS 노드 크기 조정

이전 자습서의 명령을 사용하여 Kubernetes 클러스터를 만든 경우 클러스터에 두 개의 노드가 있습니다. 이 양을 늘리거나 줄이려면 노드 수를 수동으로 조정할 수 있습니다.

다음 예제에서는 myAKSCluster라는 Kubernetes 클러스터의 노드 수를 3개로 늘립니다. 이 명령은 완료되는 데 2~3분이 걸립니다

#Azure CLI

* `az aks scale` 명령을 사용하여 클러스터 노드의 크기를 조정합니다.

```Azure CLI
az aks scale --resource-group `myResourceGroup` --name `myAKSCluster` --node-count 3
```
클러스터의 크기가 성공적으로 조정되면 다음 예와 유사하게 출력됩니다.

```출력
"aadProfile": null,
"addonProfiles": null,
"agentPoolProfiles": [
  {
    ...
    "count": 3,
    "mode": "System",
    "name": "nodepool1",
    "osDiskSizeGb": 128,
    "osDiskType": "Managed",
    "osType": "Linux",
    "ports": null,
    "vmSize": "Standard_DS2_v2",
    "vnetSubnetId": null
    ...
  }
  ...
]
```
