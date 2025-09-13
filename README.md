# MySQL Helm Chart

A simple, production-ready Helm chart for deploying MySQL on Kubernetes.

![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 8.0.40](https://img.shields.io/badge/AppVersion-8.0.40-informational?style=flat-square)

## Quick Start

### Add Helm Repository
```bash
helm repo add mysql https://yourusername.github.io/mysql/
helm repo update
```

### Install
```bash
# Basic installation
helm install my-mysql mysql/mysql

# With custom namespace
helm install my-mysql mysql/mysql -n database --create-namespace

# With custom values
helm install my-mysql mysql/mysql -f my-values.yaml
```

### Uninstall
```bash
helm uninstall my-mysql
```

## Features

- **StatefulSet** deployment with persistent storage
- **Security** contexts and secrets management
- **Health checks** with liveness and readiness probes
- **Auto scaling** with HPA support
- **Flexible configuration** via ConfigMaps and Secrets
- **Service discovery** ready
- **Backup scheduling** via CronJobs
- **MySQL 8.0** with latest features

## Configuration

### Basic Example
```yaml
# values.yaml
statefulset: true
replicaCount: 1

image:
  repository: mysql
  tag: "8.0.40"

persistence:
  enabled: true
  volumeClaimTemplates:
    - name: mysql-storage
      resources:
        requests:
          storage: 10Gi

Secrets:
  MYSQL_ROOT_PASSWORD: "your-secure-password"
  MYSQL_USER: "myuser"
  MYSQL_PASSWORD: "mypassword"
  MYSQL_DATABASE: "mydb"
```

### Production Example
```yaml
# production-values.yaml
replicaCount: 3
statefulset: true

resources:
  requests:
    memory: 2Gi
    cpu: 1000m
  limits:
    memory: 4Gi
    cpu: 2000m

persistence:
  enabled: true
  volumeClaimTemplates:
    - name: mysql-storage
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 100Gi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

nodeSelector:
  node-type: database

tolerations:
  - key: "database"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

## Access MySQL

### Get Root Password
```bash
export MYSQL_ROOT_PASSWORD=$(kubectl get secret my-mysql-secret -o jsonpath="{.data.MYSQL_ROOT_PASSWORD}" | base64 -d)
echo $MYSQL_ROOT_PASSWORD
```

### Get User Password
```bash
export MYSQL_PASSWORD=$(kubectl get secret my-mysql-secret -o jsonpath="{.data.MYSQL_PASSWORD}" | base64 -d)
echo $MYSQL_PASSWORD
```

### Port Forward
```bash
kubectl port-forward service/my-mysql-service 3306:3306
```

### Connect with MySQL Client
```bash
# As root user
mysql -h localhost -P 3306 -u root -p$MYSQL_ROOT_PASSWORD

# As regular user
mysql -h localhost -P 3306 -u myuser -p$MYSQL_PASSWORD mydb
```

### Connect from Another Pod
```bash
kubectl run -it --rm --image=mysql:8.0 --restart=Never mysql-client -- mysql -h my-mysql-service -u myuser -p
```

## Backup & Restore

### Manual Backup
```bash
# Create backup
kubectl exec -it my-mysql-0 -- mysqldump -u root -p$MYSQL_ROOT_PASSWORD --all-databases > backup.sql

# Backup specific database
kubectl exec -it my-mysql-0 -- mysqldump -u myuser -p$MYSQL_PASSWORD mydb > mydb-backup.sql
```

### Restore
```bash
# Restore all databases
kubectl exec -i my-mysql-0 -- mysql -u root -p$MYSQL_ROOT_PASSWORD < backup.sql

# Restore specific database
kubectl exec -i my-mysql-0 -- mysql -u myuser -p$MYSQL_PASSWORD mydb < mydb-backup.sql
```

### Automated Backup (CronJob)
```yaml
CronJobs:
  - name: mysql-backup
    schedule: "0 2 * * *"  # Every day at 2 AM
    image:
      repository: mysql
      tag: "8.0.40"
    command: ["sh", "-c"]
    args:
      - |
        mysqldump -h my-mysql-service -u root -p$MYSQL_ROOT_PASSWORD --all-databases > /backup/backup-$(date +%Y%m%d-%H%M%S).sql
        gzip /backup/backup-$(date +%Y%m%d-%H%M%S).sql
    env:
      - name: MYSQL_ROOT_PASSWORD
        valueFrom:
          secretKeyRef:
            name: my-mysql-secret
            key: MYSQL_ROOT_PASSWORD
    volumeMounts:
      - name: backup-storage
        mountPath: /backup
    volumes:
      - name: backup-storage
        persistentVolumeClaim:
          claimName: mysql-backup-pvc
```

## MySQL Configuration

### Custom my.cnf
```yaml
ConfigMap:
  my.cnf: |
    [mysqld]
    max_connections = 200
    innodb_buffer_pool_size = 1G
    innodb_log_file_size = 256M
    slow_query_log = 1
    slow_query_log_file = /var/log/mysql/slow.log
    long_query_time = 2

configMounts:
  - name: mysql-config
    configName: mysql-config
    path: /etc/mysql/conf.d/my.cnf
    subPath: my.cnf
```

## Configuration Reference

| Parameter | Description | Default |
|-----------|-------------|---------|
| `statefulset` | Deploy as StatefulSet instead of Deployment | `true` |
| `replicaCount` | Number of replicas | `1` |
| `image.repository` | MySQL image repository | `mysql` |
| `image.tag` | MySQL image tag | `8.0.40` |
| `persistence.enabled` | Enable persistent storage | `true` |
| `Secrets.MYSQL_ROOT_PASSWORD` | MySQL root password | `SecureRootPassword123!` |
| `Secrets.MYSQL_USER` | MySQL user | `appuser` |
| `Secrets.MYSQL_PASSWORD` | MySQL user password | `SecurePassword123!` |
| `Secrets.MYSQL_DATABASE` | MySQL database | `appdb` |
| `resources.requests.memory` | Memory request | `512Mi` |
| `resources.requests.cpu` | CPU request | `250m` |
| `autoscaling.enabled` | Enable Horizontal Pod Autoscaler | `false` |
| `service.type` | Kubernetes service type | `ClusterIP` |

See [values.yaml](values.yaml) for all available options.

## Requirements

- Kubernetes 1.19+
- Helm 3.0+
- PV provisioner support in the underlying infrastructure (for persistence)

## Common Issues

### Pod stuck in Pending
Check if you have a storage class configured:
```bash
kubectl get storageclass
```

### Connection refused
Make sure the service is running and accessible:
```bash
kubectl get svc
kubectl describe pod my-mysql-0
```

### Authentication failed
Verify the secret was created correctly:
```bash
kubectl get secret my-mysql-secret -o yaml
```

### MySQL Container Keeps Restarting
Check logs for errors:
```bash
kubectl logs my-mysql-0
kubectl describe pod my-mysql-0
```

## Performance Tuning

### InnoDB Buffer Pool Size
Set to 70-80% of available memory:
```yaml
extraEnvs:
  - name: MYSQL_INNODB_BUFFER_POOL_SIZE
    value: "2G"
```

### Connection Limits
```yaml
ConfigMap:
  my.cnf: |
    [mysqld]
    max_connections = 500
    max_user_connections = 50
```

## Development

### Local Testing
```bash
# Install locally
helm install test-mysql . -f values.yaml

# Test template rendering
helm template test-mysql . -f values.yaml

# Debug
helm install test-mysql . --dry-run --debug
```

### Contributing
Feel free to open issues or submit pull requests. This is a personal project but contributions are welcome.

## License

This chart is provided as-is under the MIT License. Use at your own risk.

---

**Note**: This is a personal Helm chart. While it's designed to be production-ready, please review and test thoroughly before using in critical environments.