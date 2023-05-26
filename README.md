# kubernetes-mysql-ex
퍼시스턴트 볼륨을 이용한 mysql 띄우기

## Mysql 용 시크릿 키 생성

주의할점 : base64로 encoding된 값을 넣어줘야한다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  password: aGtzMTM1Nzk= # hks13579
```

## Mysql 용 컨피그 맵 생성

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  host: "localhost"
  port: "3306"
  database: "jwt_security"
  user: "root"
```

## Mysql 용 PV, PVC 생성

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 5Gi # 용량 설정
  accessModes:
    - ReadWriteOnce # ReadWriteMany 같은 옵션도 존재함. (Many는 여러명 접근가능)
  persistentVolumeReclaimPolicy: Retain # Pod가 종료되어도 유지하겠다.
  storageClassName: standard # 스토리지 클래스이름
  hostPath: # 어디에 데이터를 저장할지
    path: /home/master/mysql_data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 5Gi
```

### Service, Deployment 생성

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mysql
  ports:
  - protocol: TCP
    port: 3306
    name: mysql
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0.33
          imagePullPolicy: Always
          env:
            - name: MYSQL_HOST
              value: localhost
            - name: MYSQL_PORT
              value: "3306"
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
            - name: MYSQL_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: mysql-config
                  key: database
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: mysql-config # volume 이름
              mountPath: /etc/mysql/conf.d/default_auth.cnf
              subPath: default_auth # config 이름
            - name: mysql-pvc
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-config
          configMap:
            name: mysql-config
            items:
              - key: "database"
                path: "database"
              - key: "default_auth"
                path: "default_auth"
        - name: mysql-pvc
          persistentVolumeClaim:
            claimName: mysql-pvc
```
