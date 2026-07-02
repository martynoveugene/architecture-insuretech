# Задание 2. Динамическое масштабирование контейнеров

## Часть 1. Динамическая маршрутизация на основании показателей утилизации памяти

После экспериментов остановился на следующей конфигурации:
в деплойменте:
resources -> requests -> memory: "8Mi"
resources -> limits -> memory: "10Mi"
в hpa:
averageUtilization: 70
stabilizationWindowSeconds: 60

Тест запускал с параметрами:
200 users and ramp up = 2 per sec

Дождался пока количество подов не вырастет до 4 и отключил тест.
До работающего одного пода состояние так и не вернулось.

Скриншоты в директории [screenshots-mem](screenshots-mem)

## Часть 2. Динамическая маршрутизация на основании показателей количества запросов в секунду
Установка прометеуса
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts                                                                                    
helm repo update
kubectl create namespace monitoring
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring
```

Настраиваем сбор сырых метрик из подов приложения: 
```
kubectl apply -f .\pod-monitor.yaml
```

Установка/обновление адаптера для подсчета http_requests_per_second
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install my-adapter prometheus-community/prometheus-adapter --namespace monitoring -f .\adapter-values.yaml

helm upgrade my-adapter prometheus-community/prometheus-adapter --namespace monitoring -f .\adapter-values.yaml
```

Проверка, что расчетная метрика появилась:
```
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1"

kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second"
```

Если нет, смотрим логи:
```
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus-adapter --tail=50
```

Удаляем предыдущую HPA политику
```
kubectl delete hpa mem-hpa -n default
kubectl apply -f .\MemDeployment.yaml
```

Применяем новую HPA политику по rps:
```
kubectl apply -f .\MemHpaRps.yaml
kubectl get hpa mem-hpa-rps
```

Даем возрастающую нагрузку и смотрим как прибывают поды,
убираем нагрузку и количество подов уменьшается: 
```
kubectl get events --field-selector reason=SuccessfulRescale
```

Скриншоты в директории [screenshots-rps](screenshots-rps)

