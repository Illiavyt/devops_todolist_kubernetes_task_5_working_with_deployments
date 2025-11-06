# Як розгорнути застосунок у Kubernetes

# Створити namespace:
kubectl create namespace mateapp

# Застосувати Deployment:
kubectl apply -f deployment.yml

# Застосувати Horizontal Pod Autoscaler:
kubectl apply -f hpa.yml

# Перевірити створені ресурси:
kubectl get all -n mateapp

# Якщо потрібно відкрити доступ до застосунку, створити Service:
apiVersion: v1
kind: Service
metadata:
  name: todoapp-service
  namespace: mateapp
spec:
  selector:
    app: todoapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: NodePort

# І застосувати:
kubectl apply -f service.yml

# Отримати адресу доступу:
minikube service todoapp-service -n mateapp

# або, якщо працюєш у кластері, дізнатися NodePort:
kubectl get svc -n mateapp

# Пояснення вибору ресурсів
resources:
  requests:
    cpu: "250m"
    memory: "64Mi"
  limits:
    cpu: "500m"
    memory: "128Mi"
# CPU 250m / Memory 64Mi у requests означає, що кожен pod отримає мінімальний гарантований обсяг ресурсів, достатній для Django у стані простою.
# CPU 500m / Memory 128Mi у limits — це обмеження, щоб один pod не перевантажував вузол при пікових запитах.
# Такий баланс дозволяє запускати щонайменше 2 поди навіть на невеликих кластерах.

# Пояснення конфігурації HPA
minReplicas: 2
maxReplicas: 5
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
# minReplicas: 2 — забезпечує високу доступність: навіть якщо один pod оновлюється або падає, другий продовжує обслуговувати трафік.
# maxReplicas: 5 — обмежує масштабування, щоб не створювати надлишкові ресурси при пікових навантаженнях.
# CPU 70% і Memory 70% як пороги — це типовий компроміс між продуктивністю та стабільністю. HPA збільшує кількість podів, якщо середнє навантаження перевищує ці значення.

# Пояснення стратегії розгортання
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
# RollingUpdate — безперервне оновлення подів без простою.
# maxUnavailable: 1 — під час оновлення максимум один pod може бути недоступним.
# maxSurge: 1 — Kubernetes може створити один додатковий pod, щоб зберегти потрібну кількість працюючих екземплярів.
# Ці параметри гарантують плавне оновлення застосунку без перерв у роботі користувачів.

# Як отримати доступ до застосунку
# Після створення сервісу (NodePort або LoadBalancer):
# Якщо ти працюєш локально (наприклад, через Minikube):
minikube service todoapp-service -n mateapp
# відкриє браузер із URL виду:
http://127.0.0.1:xxxxx
# Якщо кластер у хмарі (GKE, EKS, AKS), тип сервісу можна змінити на LoadBalancer:
type: LoadBalancer
# І тоді доступ буде через зовнішню IP-адресу:
kubectl get svc todoapp-service -n mateapp