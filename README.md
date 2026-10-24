# VProfile on Kubernetes (AWS Kops)

A hands-on DevOps project to containerize and deploy the **VProfile Java web application** on a Kubernetes cluster provisioned in AWS with **Kops**. This repository brings together the original application, Docker build configuration, and Kubernetes manifests for a multi-tier application.

> **Project status (24 October 2026):** Kops reports one control-plane instance and two worker instances as `UpToDate`, and the Kubernetes API hostname resolves in DNS. Kubernetes node readiness, application deployment, and end-to-end browser access are **not yet documented as verified**. Add actual verification output after testing.

## 1. Architecture

```text
                         External users
                               |
                               v
                  DNS / public ingress endpoint
                               |
                               v
                    Ingress Controller (*required*)
                               |
                               v
                     Ingress routing rules
                               |
                               v
                 vproapp-service (ClusterIP :8080)
                               |
                               v
                     Tomcat / VProfile Pod
                         /      |       \
                        v       v        v
                vprodb:3306  vprocache01  vprormq01
                    MySQL     :11211       :5672
                      |
                      v
                PVC / AWS EBS storage
```

Ingress exposes the HTTP application only. MySQL, Memcached, and RabbitMQ communicate with Tomcat using internal ClusterIP Services. A functioning, maintained Ingress Controller and external exposure method must be configured separately.

## 2. Technologies

| Layer | Technology | Responsibility |
| --- | --- | --- |
| Web application | Java, Spring MVC, Apache Tomcat | Serves VProfile |
| Database | MySQL | Persistent application data |
| Cache | Memcached | Caching |
| Messaging | RabbitMQ | Messaging |
| Build | Maven (`pom.xml`) | Java application build |
| Containers | Docker / Docker Hub | Application and dependency images |
| Orchestration | Kubernetes | Deployments, Pods and Services |
| Infrastructure | AWS EC2, Kops, VPC, Auto Scaling Groups | Kubernetes infrastructure |
| State & DNS | Amazon S3, Route 53 | Kops state and cluster DNS |
| Persistent data | PersistentVolumeClaim / EBS CSI | MySQL volume |
| Web routing | Kubernetes Ingress + controller | HTTP ingress routing |

## 3. Cluster Configuration

| Setting | Lab configuration |
| --- | --- |
| AWS region | `us-east-1` |
| Cluster DNS name | `kubevpro.achaudhry.online` |
| Kops state bucket | `s3://kops0193` |
| Availability zones | `us-east-1a`, `us-east-1b` |
| Control plane | 1 × `t3.medium` |
| Worker nodes | 2 × `t3.small` |
| Management instance | Separate Ubuntu EC2 (not a cluster node) |

This is a **learning/lab architecture**, not a highly available production control plane. AWS instances, disks, IPv4 addresses and other resources can incur charges while the lab runs.

## 4. Repository Contents

```text
vprokube/
├── kubedefs/              # Kubernetes Deployment, Service, PVC, Secret and Ingress definitions
├── Docker-files/          # Docker build definitions
├── docker-compose.yml     # Container-based local workflow
├── src/                   # VProfile application source code
├── pom.xml                # Maven build configuration
└── README.md              # This guide
```

The following Kubernetes manifest filenames are used in this guide. Confirm they match the repository files before running commands:

```text
kubedefs/appdeploy.yaml       kubedefs/appservice.yaml
kubedefs/appingress.yaml     kubedefs/dbdeploy.yaml
kubedefs/dbservice.yaml      kubedefs/dbpvc.yaml
kubedefs/mcdep.yaml          kubedefs/mcservice.yaml
kubedefs/rmqdeploy.yaml      kubedefs/rmqservice.yaml
```

**Security:** Do not commit real `secret.yaml` files, AWS credentials, private SSH keys, kubeconfigs containing client keys, or hardcoded passwords. Kubernetes Secret data encoded as Base64 is **not encrypted**. If credentials were previously committed, remove exposure, rotate the credentials, and review repository history.

## 5. Kubernetes Design

- **Deployments:** Manage Tomcat, MySQL, Memcached and RabbitMQ Pods.
- **ClusterIP Services:** Provide stable DNS names and ports to application dependencies.
- **MySQL PVC:** Requests persistent storage; requires a working StorageClass and volume provisioner.
- **Secret:** Supplies database and RabbitMQ authentication values (`db-pass` and `rmq-pass`).
- **Tomcat init containers:** Check that MySQL and Memcached Service names resolve before starting Tomcat. DNS resolution alone does not establish service readiness.
- **Ingress:** Routes web requests to `vproapp-service`; requires the correct `IngressClass` and controller.

### Internal Service Ports

| Service | Port | Deployment/backend |
| --- | ---: | --- |
| `vproapp-service` | `8080` | Tomcat / VProfile |
| `vprodb` | `3306` | MySQL |
| `vprocache01` | `11211` | Memcached |
| `vprormq01` | `5672` | RabbitMQ |

The names must match application configuration (`application.properties`) and Service selectors must match the corresponding Pod labels.

## 6. Prerequisites and Predeployment Checks

1. AWS credentials with permissions required by Kops, and access to the intended S3 state bucket and Route 53 DNS zone.
2. A healthy Kops cluster and working `kubectl` credentials.
3. An EBS CSI driver and usable StorageClass for the MySQL PVC.
4. A maintained Ingress Controller with a matching `IngressClass`; external network path and application DNS configured.
5. Appropriate credentials and container images; application configuration must match Kubernetes Service names.

Run from the **Kops management EC2** after connecting to the cluster:

```bash
export NAME=kubevpro.achaudhry.online
export KOPS_STATE_STORE=s3://kops0193

kops validate cluster --name="$NAME" --state="$KOPS_STATE_STORE" --wait 10m
kubectl get nodes -o wide
kubectl get storageclass
kubectl get ingressclass
kubectl get pods -A
```

**Check before applying:** The course's `storageClassName: standard` is not guaranteed to exist on AWS Kops. Choose a confirmed StorageClass, and ensure the PVC and EBS volume topology work with the node's availability zone. Also verify the RabbitMQ Service selector and named `targetPort` match its Deployment. The default RabbitMQ `guest` account generally cannot authenticate from a different Pod; use a dedicated user and align the application configuration.

## 7. Deploy the Application

The following assumes the repository has been cloned to a machine with Kubernetes access and the manifests have been validated. Apply everything in the **same namespace** unless the manifests explicitly account for cross-namespace access.

### A. Create credentials securely

Create a Secret named `app-secret` with keys `db-pass` and `rmq-pass`, using a secure credential management process. Do not embed genuine passwords in tracked YAML or commit your shell history containing passwords.

Verify the Secret exists without printing its values:

```bash
kubectl get secret app-secret
```

### B. Create database storage and backends

```bash
kubectl apply -f kubedefs/dbpvc.yaml
kubectl apply -f kubedefs/dbdeploy.yaml
kubectl apply -f kubedefs/dbservice.yaml

kubectl apply -f kubedefs/mcdep.yaml
kubectl apply -f kubedefs/mcservice.yaml

kubectl apply -f kubedefs/rmqdeploy.yaml
kubectl apply -f kubedefs/rmqservice.yaml
```

Check backend status before deploying Tomcat:

```bash
kubectl get pvc
kubectl get pods -o wide
kubectl get svc
kubectl get endpointslice
```

### C. Deploy Tomcat and its Service

```bash
kubectl apply -f kubedefs/appdeploy.yaml
kubectl apply -f kubedefs/appservice.yaml
kubectl rollout status deployment/vproapp --timeout=5m
```

### D. Configure and apply Ingress

Confirm `appingress.yaml` uses the **installed** controller's `ingressClassName`. Do not assume the older course's community ingress-nginx controller is suitable for new environments. For a simple `/` Prefix route, a regex annotation is unnecessary.

```bash
kubectl apply -f kubedefs/appingress.yaml
kubectl get ingress
kubectl describe ingress vproapp-ingress
```

The application hostname in the supplied manifest is `vprofile.achaudhry.online`; make sure its DNS points to the actual public ingress endpoint, not merely the Kubernetes API address.

## 8. Validation and Troubleshooting

```bash
# Workloads and network
kubectl get deployments
kubectl get pods -o wide
kubectl get svc
kubectl get ingress
kubectl get endpointslice

# Storage
kubectl get pvc
kubectl describe pvc db-pv-claim

# Deployment progress
kubectl rollout status deployment/vprodb --timeout=5m
kubectl rollout status deployment/vproapp --timeout=5m

# Investigate a problem
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> -c init-mydb
```

Typical failure points include a PVC stuck in `Pending`, a Service selector that matches no Pods, incorrect target port names, RabbitMQ guest-user restrictions, an absent Ingress Controller, invalid DNS, and a Pod stuck in an init container waiting for service discovery.

**End-to-end success criteria:** Three Kubernetes nodes `Ready`; backend and Tomcat Pods healthy; MySQL PVC `Bound`; expected Service endpoints present; DNS correctly routed; and the VProfile web application reachable and functioning from a browser. Record actual command outputs or screenshots when achieved.

## 9. Lab Cleanup and AWS Cost Control

When the deployment has been tested and data is no longer needed, inspect storage and backups and then use Kops to clean up the temporary cluster:

```bash
kops delete cluster --name="$NAME" --state="$KOPS_STATE_STORE"
# Execute the destructive command below only after reviewing the preview:
kops delete cluster --name="$NAME" --state="$KOPS_STATE_STORE" --yes
```

Cluster deletion can destroy application data and may leave separately created billable resources. Check EC2, EBS, Load Balancers, public IPv4 allocations, Route 53, S3 and AWS Billing after cleanup. Keep the state bucket and DNS zone only if you plan to reuse them.

## 10. Learning Outcomes and Attribution

This exercise covers the Kubernetes control plane/worker model, workload orchestration, Service discovery, named ports, volumes and PVCs, initialization ordering, Ingress, and AWS Kops provisioning and troubleshooting.

**Attribution:** Built as a DevOps learning exercise using the VProfile application and course baseline by its original authors. The repository is meant to document hands-on Kubernetes/AWS setup, configuration and verification; original source code and course material should not be presented as solely original work.

**Evidence to add after deployment:** sanitized output of `kubectl get nodes`, `kubectl get pods`, `kubectl get pvc`, `kubectl get ingress`, successful application browser access, and a brief list of issues resolved.
