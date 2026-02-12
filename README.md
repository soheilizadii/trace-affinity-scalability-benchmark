# Trace-Affinity Scalability Benchmark

This repository contains the research artifact for my Master’s thesis. It provides the deployment configurations and experiment definitions required to benchmark a trace-processing pipeline that preserves **trace affinity** while scaling the **otel-transformer** horizontally. The benchmark is executed with **Theodolite** on Kubernetes and can be visualized in **ExplorViz**.

Running the benchmark typically involves these steps:

1. **Deploy the benchmarked system** to Kubernetes: TeaStore (instrumented with Kieker), the Kieker–Kafka Bridge, Kafka, the otel-transformer, and the OpenTelemetry Collector.
2. **Deploy ExplorViz**  to visualize the produced OpenTelemetry traces and validate end-to-end observability.
3. **Run Theodolite experiments** by creating executions that define the workload (e.g., JMeter user levels) and the scale-out configuration (e.g., 1–3 otel-transformer replicas) together with the throughput/correctness SLOs.
4. **Collect and inspect results**, including Prometheus/Grafana metrics and Theodolite outputs.

## Prerequisites 

To get started, you need: 

* A running Kubernetes cluster (for testing purposes, you might want to use [Minikube](https://minikube.sigs.k8s.io/), [kind](https://kind.sigs.k8s.io/) or [k3d](https://k3d.io/))
* [Helm installed](https://helm.sh/) on you local machine.

## Cluster Preparation

Before running a benchmark, we need to install Theodolite and Istio on our cluster.

### Install Theodolite

In general, Theodolite can be installed using Helm.

```sh
helm repo add theodolite https://www.theodolite.rocks
helm repo update
```
run the following command to install Theodolite:
```sh
helm install theodolite theodolite/theodolite
```

### Create ConfigMaps containing all components

To create a ConfigMap containing  OTel transformer resources, simply run:

```sh
kubectl create configmap  otel-transformer-deployment --from-file=kubernetes/otel-transformer/
```

Likewise, we have to create a ConfigMap for the JMeter profile and a ConfigMap containing a JMeter deployment:

```sh
kubectl create configmap teastore-jmeter-browse --from-file=./Jmeter/teastore_browse_nogui.jmx
kubectl create configmap teastore-jmeter-deployment --from-file=./Theodolite-config/jmeter.yaml
```

## Deploy TeaStore and the Kieker–Kafka Bridge in the Kubernetes Cluster  (Helm)

Run the following command to deploy TeaStore as the workload system together with the Kieker–Kafka Bridge using the provided Helm chart:
```sh
helm install teastore-observability ./Helm/teastore-observability-chart 
```

### Deploy ExplorViz in the Kubernetes Cluster (Helm)

To visualize the behavior of TeaStore while the benchmark is running, we also
deploy ExplorViz in the same Kubernetes cluster.
Helm chart:

```sh
helm install explorviz ./Helm/explorviz-chart
```
Once all pods are running, the ExplorViz frontend can be accessed via:
```sh 
kubectl port-forward svc/frontend 8082:80
```

### Start the Benchmark

To let Prometheus scrape metrics, we need to create a PodMonitor for OTel transformer.

apply the PodMonitor as follows:

```sh
kubectl apply -f ./Theodolite-config/otel-transformer-servicemonitor.yaml
```

Next, we need to deploy our benchmark:

```sh
kubectl apply -f ./Theodolite-config/benchmark.yaml
```

To now start benchmark execution, we deploy our Execution resource defined previously:

```sh
kubectl apply -f ./Theodolite-config/execution-users.yaml
```

## Accessing Benchmark Results 

Theodolite comes with a results access sidecar. It allows to copy all benchmark results from the Theodolite pod to your current working directory on your host machine with the following command:

```sh
kubectl cp $(kubectl get pod -l app=theodolite -o jsonpath="{.items[0].metadata.name}"):results . -c results-access
```

## Visualize Metrics in Grafana (Theodolite stack)

Theodolite deploys Prometheus and Grafana (via its Helm chart). After installation, you can access Grafana locally via port-forward:

```sh
kubectl port-forward -n theodolite svc/theodolite-grafana 8083:80
```
