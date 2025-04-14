---
# 🧩 Desafio Técnico - Analista de Infraestrutura

Projeto desenvolvido em um **Macbook Air com Apple Silicon**, utilizando o **Docker Desktop** para o ambiente local e o **Minikube** para provisionamento do cluster Kubernetes.

---

## ⚙️ Etapa 1 - Docker: Jenkins + Tomcat + Jolokia

### Objetivo
Subir o Jenkins dentro de um contêiner Docker com o Tomcat como servidor de aplicação e com o agente Jolokia habilitado para exposição de métricas.

### Estrutura utilizada

**Pasta:** `jenkins-tomcat-jolokia`

**Dockerfile:**
```dockerfile
FROM tomcat:11.0-jdk21

RUN curl -fL -o /usr/local/tomcat/webapps/jenkins.war \
    https://get.jenkins.io/war-stable/latest/jenkins.war || \
    { echo "❌ Falha ao baixar Jenkins WAR"; exit 1; }

RUN mkdir -p /opt && \
    curl -fL -o /opt/jolokia-jvm-agent.jar \
    "https://search.maven.org/remotecontent?filepath=org/jolokia/jolokia-agent-jvm/2.2.8/jolokia-agent-jvm-2.2.8-javaagent.jar" || \
    { echo "❌ Falha ao baixar Jolokia Agent"; exit 1; }

ENV JENKINS_HOME=/var/jenkins_home \
    CATALINA_OPTS="-Djava.awt.headless=true -javaagent:/opt/jolokia-jvm-agent.jar=port=8778,host=0.0.0.0"

VOLUME /var/jenkins_home
RUN mkdir -p /var/jenkins_home && chmod 755 /var/jenkins_home

EXPOSE 8080 8778

HEALTHCHECK --interval=30s --timeout=10s --start-period=120s --retries=3 \
    CMD curl -f http://localhost:8080/jenkins || exit 1

CMD ["catalina.sh", "run"]
```

### Execução
```bash
cd jenkins-tomcat-jolokia
docker build -t jenkins-tomcat-jolokia .
docker run -d -p 8080:8080 -p 8778:8778 --name jenkins-container jenkins-tomcat-jolokia
```

- Acesse o Jenkins: http://localhost:8080/jenkins
- Métricas Jolokia: http://localhost:8778/jolokia

---

## ☸️ Etapa 2 - Kubernetes

### Objetivo
Recriar a estrutura da Etapa 1 em um cluster Kubernetes local usando Minikube.

**Pasta:** `k8s`

### Arquivos criados:

#### `jenkins-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      initContainers:
      - name: download-jolokia
        image: curlimages/curl
        command: ['sh', '-c', 'curl -L -o /shared/jolokia-jvm-agent.jar https://search.maven.org/remotecontent?filepath=org/jolokia/jolokia-agent-jvm/2.2.8/jolokia-agent-jvm-2.2.8-javaagent.jar']
        volumeMounts:
        - name: shared-data
          mountPath: /shared
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts-jdk11
        env:
        - name: JAVA_OPTS
          value: "-Djava.awt.headless=true -javaagent:/opt/jolokia/jolokia-jvm-agent.jar=port=8778,host=0.0.0.0"
        ports:
        - containerPort: 8080
        - containerPort: 8778
        volumeMounts:
        - name: jenkins-data
          mountPath: /var/jenkins_home
        - name: shared-data
          mountPath: /opt/jolokia
      volumes:
      - name: jenkins-data
        emptyDir: {}
      - name: shared-data
        emptyDir: {}
```

#### `jenkins-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  type: NodePort
  ports:
  - name: http-web
    port: 8080
    targetPort: 8080
    nodePort: 30080
    protocol: TCP
  - name: jolokia-metrics
    port: 8778
    targetPort: 8778
    protocol: TCP
  selector:
    app: jenkins
```

### Comandos
```bash
minikube start
kubectl apply -f k8s/jenkins-deployment.yaml
kubectl apply -f k8s/jenkins-service.yaml
kubectl port-forward svc/jenkins 8080:8080
kubectl port-forward svc/jenkins 8778:8778
```

### Acesso via port-forward
- Jenkins: http://localhost:8080
- Jolokia: http://localhost:8778/jolokia

---

## 📊 Etapa 3 - Monitoramento com Prometheus e Grafana

### Objetivo
Coletar métricas do Jenkins via Jolokia e do cluster via Node Exporter, usando Prometheus e Grafana.

### Arquivo único: `monitoring.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: default
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s

    scrape_configs:
      - job_name: 'jenkins-jolokia'
        metrics_path: '/jolokia'
        static_configs:
          - targets: ['jenkins:8778']
            labels:
              service: 'jenkins'

      - job_name: 'node-exporter'
        static_configs:
          - targets: ['node-exporter:9100']
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus/prometheus.yml
          subPath: prometheus.yml
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-config
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 30090
  selector:
    app: prometheus
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
      volumes:
      - name: grafana-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30300
  selector:
    app: grafana
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: default
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
---
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: default
spec:
  ports:
  - port: 9100
    targetPort: 9100
  selector:
    app: node-exporter
  clusterIP: None
```

### Comandos
```bash
kubectl apply -f k8s/monitoring.yaml
kubectl port-forward svc/prometheus 9090:9090
kubectl port-forward svc/grafana 3000:3000
```

### Acesso via port-forward
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000

---

## ✅ Conclusão

Projeto completo, com todas as etapas concluídas:
- Jenkins em Tomcat com Jolokia em Docker
- Kubernetes com deployment completo via Minikube
- Monitoramento com Prometheus e Grafana

Documentação feita com base nas boas práticas e testes realizados localmente com sucesso.

---

## 👨‍💻 Tecnologias e Ferramentas Utilizadas
- Macbook Air M4
- Docker Desktop
- Minikube
- Jenkins
- Tomcat
- Jolokia
- Prometheus
- Grafana
- Node Exporter

---

## 🎥 Vídeo de Apresentação
**Gravado com Loom** — [[Vídeo de apresentação do desafio](https://www.loom.com/share/1ffb2ec4828b4146a6ccfe3aa70b7fa9?sid=ebcb7402-12c3-4dc3-bf27-88c1453f9cac)]

---

## 👤 Autor
Marcos Alves — [github.com/mviniciius29](https://github.com/mviniciius29)
