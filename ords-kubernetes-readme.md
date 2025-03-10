# ORDS on Minikube with Oracle 19c Database

This documentation explains how to deploy Oracle REST Data Services (ORDS) on Minikube that connects to an Oracle 19c database running in Docker.

## Architecture Overview

The setup consists of:
1. Oracle 19c database running in Docker on the host machine
2. ORDS running in Kubernetes (Minikube) that connects to the Oracle database
3. Ingress configuration to access ORDS from the host machine

## Kubernetes Resource Files

### 1. ords-pvc.yaml
This file defines a Persistent Volume Claim to store ORDS configuration.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ords-config-pvc
  namespace: ords-zoo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

- Creates a 1GB persistent volume claim named `ords-config-pvc` in the `ords-zoo` namespace
- Uses `ReadWriteOnce` access mode which allows the volume to be mounted as read-write by a single node

### 2. ords-secret.yaml
This file defines credentials and connection details for the Oracle database.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ords-secret
  namespace: ords-zoo
type: Opaque
stringData:
  admin_user: sys
  admin_pwd: MyDBPassword  # Matches ORACLE_PWD from docker run command
  proxy_pwd: oracle
  db_host: 192.168.49.1  # Special hostname to access host from minikube
  db_port: "1521"
  db_servicename: ORCLPDB1
```

- Creates a Kubernetes secret to store sensitive database connection information
- `192.168.49.1` is the special IP address that allows pods in Minikube to access services on the host machine

### 3. ords-service.yaml
This file defines a Kubernetes service to expose the ORDS deployment.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ords-nondev
  namespace: ords-zoo
spec:
  selector:
    app.kubernetes.io/instance: ords-nondev
    app.kubernetes.io/name: ords-nondev
  ports:
    - name: http-ords
      port: 8888
      protocol: TCP
      targetPort: 8080
  type: ClusterIP
```

- Creates a ClusterIP service that exposes port 8888 and routes to port 8080 on the ORDS pods
- Selects pods with matching labels for `app.kubernetes.io/instance` and `app.kubernetes.io/name`

### 4. ords-configmap.yaml / debug-configmap.yaml
These files define initialization scripts for ORDS. The debug version includes additional diagnostics.

**Standard version (`ords-configmap.yaml`):**
- Connects to the database as SYSDBA
- Installs ORDS with proper connection parameters
- Sets up APEX files by downloading and configuring them
- Configures the APEX static path

**Debug version (`debug-configmap.yaml`):**
- Includes additional connectivity tests
- Displays environment variables
- Tests database connectivity with timeout
- Runs tnsping and sqlplus tests if available
- Provides detailed error information if installation fails

### 5. ords-deployment.yaml
This file defines the ORDS deployment with initialization container.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ords-nondev
  namespace: ords-zoo
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/instance: ords-nondev
      app.kubernetes.io/name: ords-nondev
  template:
    metadata:
      labels:
        app.kubernetes.io/instance: ords-nondev
        app.kubernetes.io/name: ords-nondev
    spec:
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: ords-config-pvc
        - name: scripts
          configMap:
            name: ords-init-config
            defaultMode: 0755
      initContainers:
        - name: set-config
          image: container-registry.oracle.com/database/ords:latest
          # ... env and volume mounts ...
      containers:
        - name: app
          image: container-registry.oracle.com/database/ords:latest
          command:
            - ords
            - serve
          # ... ports and volume mounts ...
```

Key components:
- Uses an init container (`set-config`) to run the ORDS setup script before starting the main container
- Main container runs `ords serve` to start the ORDS server
- Both containers use the official Oracle ORDS image
- Environment variables are populated from the Kubernetes secret
- Configuration is stored on the persistent volume
- Setup script is mounted from the ConfigMap

### 6. ords-ingress.yaml
This file defines an Ingress resource to access ORDS from outside the cluster.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ords-ingress
  namespace: ords-zoo
spec:
  rules:
  - host: ords.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ords-nondev
            port:
              number: 8888
```

- Creates an Ingress resource that routes traffic from `ords.local` to the ORDS service
- Requires adding `127.0.0.1 ords.local` to your local hosts file

## Step-by-Step Implementation Guide

### 1. Set Up Oracle Database in Docker

```bash
docker run -d \
  --name external-oracle-19c \
  -p 1521:1521 \
  -e ORACLE_PWD=MyDBPassword \
  oracledb19c/oracle.19.3.0-ee:oracle19.3.0-ee
```

This starts an Oracle 19c database in Docker with:
- Port 1521 exposed to the host
- Default password set to "MyDBPassword"
- Container named "external-oracle-19c"

### 2. Set Up Kubernetes Resources

1. Create the namespace:
```bash
kubectl create namespace ords-zoo
```

2. Apply all the Kubernetes resource files:
```bash
kubectl apply -f ords-pvc.yaml
kubectl apply -f ords-secret.yaml
kubectl apply -f ords-configmap.yaml  # or debug-configmap.yaml for troubleshooting
kubectl apply -f ords-deployment.yaml
kubectl apply -f ords-service.yaml
kubectl apply -f ords-ingress.yaml
```

3. Check that all resources are created successfully:
```bash
kubectl get all -n ords-zoo
```

4. Check the logs of the init container for any errors:
```bash
kubectl logs -n ords-zoo deploy/ords-nondev -c set-config
```

### 3. Set Up Database Schema and REST Endpoints

1. Connect to the Oracle database as SYS:
```sql
-- Connect as SYS (SYSDBA)
-- Grant inheritance privileges
GRANT INHERIT PRIVILEGES ON USER SYS TO ORDS_METADATA;
GRANT INHERIT PRIVILEGES ON USER SYS TO RESTDEMO;

-- Grant connect privileges to ORDS_PUBLIC_USER (if needed)
BEGIN
  ORDS_ADMIN.ENABLE_SCHEMA(
    p_enabled             => TRUE,
    p_schema              => 'RESTDEMO',
    p_url_mapping_type    => 'BASE_PATH',
    p_url_mapping_pattern => 'restdemo',
    p_auto_rest_auth      => FALSE
  );
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
END;
/
```

2. Create test data in the RESTDEMO schema:
```sql
-- Connected as RESTDEMO
CREATE TABLE sample_data (
  id NUMBER PRIMARY KEY,
  name VARCHAR2(100),
  description VARCHAR2(200)
);

INSERT INTO sample_data VALUES (1, 'Test Item', 'This is a test');
COMMIT;
```

### 4. Access the REST API

Access the REST endpoint using the Minikube IP and port:
```
http://127.0.0.1:53660/ords/restdemo/sample_data/
```

Or using the Ingress (if configured):
```
http://ords.local/ords/restdemo/sample_data/
```

## Troubleshooting

If you encounter connection issues between ORDS and Oracle:

1. Verify the database is accessible from the host:
```bash
docker exec -it external-oracle-19c sqlplus sys/MyDBPassword@ORCLPDB1 as sysdba
```

2. Check that the IP address in `ords-secret.yaml` is correct:
   - For Minikube, the host IP is typically `192.168.49.1`
   - Verify with `ip addr | grep -A2 "eth0\|enp0s"`

3. Use the debug ConfigMap for more detailed logging:
```bash
kubectl apply -f debug-configmap.yaml
kubectl rollout restart deployment/ords-nondev -n ords-zoo
```

4. Check the logs for detailed connection information:
```bash
kubectl logs -n ords-zoo deploy/ords-nondev -c set-config
```

## Additional Notes

1. This setup uses a special IP address (`192.168.49.1`) to allow pods in Minikube to access the host machine where the Oracle database is running.

2. ORDS authentication is not enabled in this example. For production, you should configure proper authentication.

3. The Oracle service name (`ORCLPDB1`) may vary depending on your Oracle installation. Adjust it in the secret if needed.

4. The ingress hostname (`ords.local`) needs to be added to your hosts file to work properly.

5. For production use, secure your credentials properly and consider using a secrets management solution.
