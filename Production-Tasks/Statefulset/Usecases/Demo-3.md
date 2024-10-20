### End-to-End Implementation of Apache Kafka Using StatefulSets

In this implementation, we will deploy **Apache Kafka** using **StatefulSets** in Kubernetes. Kafka is widely used for building real-time data pipelines and streaming applications, and using StatefulSets helps manage the complexities of stateful applications like Kafka brokers.

### Step 1: Prerequisites

- **Kubernetes Cluster**: Ensure you have a running Kubernetes cluster.
- **kubectl**: Ensure `kubectl` is configured properly.

### Step 2: Create a Namespace

Create a dedicated namespace for your Kafka deployment.

```bash
kubectl create namespace kafka-cluster
```

### Step 3: Create Zookeeper StatefulSet

Kafka requires Zookeeper to manage brokers. We'll deploy Zookeeper first.

Create a file named `zookeeper-statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zookeeper
  namespace: kafka-cluster
spec:
  serviceName: "zookeeper"
  replicas: 3  # Number of Zookeeper nodes
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      containers:
      - name: zookeeper
        image: wurstmeister/zookeeper:3.4.6
        ports:
        - containerPort: 2181
        env:
        - name: ZOO_MY_ID
          value: "1"  # Set Zookeeper ID for the node
        volumeMounts:
        - name: zookeeper-storage
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: zookeeper-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

### Step 4: Create Zookeeper Headless Service

Create a file named `zookeeper-service.yaml` for the headless service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: zookeeper
  namespace: kafka-cluster
spec:
  clusterIP: None  # Headless service
  selector:
    app: zookeeper
  ports:
  - port: 2181
    targetPort: 2181
```

### Step 5: Apply Zookeeper Configuration

Run the following commands to create the Zookeeper StatefulSet and service:

```bash
kubectl apply -f zookeeper-statefulset.yaml
kubectl apply -f zookeeper-service.yaml
```

### Step 6: Verify Zookeeper Deployment

1. **Check Zookeeper StatefulSet**:

```bash
kubectl get statefulsets -n kafka-cluster
```

2. **Check Zookeeper Pods**:

```bash
kubectl get pods -n kafka-cluster
```

You should see Pods named `zookeeper-0`, `zookeeper-1`, and `zookeeper-2`.

### Step 7: Create Kafka StatefulSet

Next, create a StatefulSet for Kafka.

Create a file named `kafka-statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: kafka-cluster
spec:
  serviceName: "kafka"
  replicas: 3  # Number of Kafka brokers
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: wurstmeister/kafka:latest
        ports:
        - containerPort: 9092
        env:
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "PLAINTEXT://kafka-0.kafka.kafka-cluster.svc.cluster.local:9092"
        - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
          value: "PLAINTEXT:PLAINTEXT"
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: "zookeeper:2181"
        - name: KAFKA_BROKER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        volumeMounts:
        - name: kafka-storage
          mountPath: /kafka
  volumeClaimTemplates:
  - metadata:
      name: kafka-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

### Step 8: Create Kafka Headless Service

Create a file named `kafka-service.yaml` for the Kafka service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka
  namespace: kafka-cluster
spec:
  clusterIP: None  # Headless service
  selector:
    app: kafka
  ports:
  - port: 9092
    targetPort: 9092
```

### Step 9: Apply Kafka Configuration

Run the following commands to create the Kafka StatefulSet and service:

```bash
kubectl apply -f kafka-statefulset.yaml
kubectl apply -f kafka-service.yaml
```

### Step 10: Verify Kafka Deployment

1. **Check Kafka StatefulSet**:

```bash
kubectl get statefulsets -n kafka-cluster
```

2. **Check Kafka Pods**:

```bash
kubectl get pods -n kafka-cluster
```

You should see Pods named `kafka-0`, `kafka-1`, and `kafka-2`.

### Step 11: Access Kafka

To produce or consume messages from Kafka, you can use the Kafka CLI tools. First, port-forward to one of the Kafka brokers:

```bash
kubectl port-forward kafka-0 9092:9092 -n kafka-cluster
```

### Step 12: Produce and Consume Messages

1. **Produce a Message**:

You can use the following command to produce messages:

```bash
kubectl exec -it kafka-0 -n kafka-cluster -- kafka-console-producer.sh --broker-list localhost:9092 --topic test
```

Then type messages into the terminal and press Enter.

2. **Consume Messages**:

In another terminal, use the following command to consume messages:

```bash
kubectl exec -it kafka-0 -n kafka-cluster -- kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
```

You should see the messages you produced in real-time.

### Step 13: Monitor Kafka Cluster

You can monitor your Kafka cluster using various tools, such as **Kafka Manager**, **Prometheus**, or **Grafana**.

### Step 14: Cleanup Resources

To delete the StatefulSet and associated resources:

```bash
kubectl delete -f kafka-statefulset.yaml
kubectl delete -f kafka-service.yaml
kubectl delete -f zookeeper-statefulset.yaml
kubectl delete -f zookeeper-service.yaml
kubectl delete namespace kafka-cluster
```

### Conclusion

In this implementation:
- We deployed an Apache Kafka cluster using StatefulSets.
- Configured Zookeeper to manage Kafka brokers.
- Verified the setup by producing and consuming messages.

This setup provides a robust messaging system suitable for applications that require real-time data processing and streaming capabilities. If you have any further questions or need assistance, feel free to ask!
