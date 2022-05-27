## mysql2postgresql-jdbc-agent

- 使用`java agent`无侵入的方式使用`postgresql`替换`mysql`. 参考 `https://github.com/wuwen5/ojdbc-mysql2oracle` 移植了`postgresql`版本
- `nacos`基于`2.1.0` 测试
- `xxljob`基于`2.3.0` 测试

### nacos 配置
- 修改启动参数增加agent
```shell
#将JAVA_OPT="${JAVA_OPT} -jar ${BASE_DIR}/target/${SERVER}.jar" 改为
JAVA_OPT="${JAVA_OPT} -javaagent:${BASE_DIR}/target/mysql2postgresql-jdbc-agent-1.0.0.jar -jar ${BASE_DIR}/target/${SERVER}.jar" 
```

## docker 

```
docker build -t mysql2postgresql-jdbc-agent:1.0.0 .
```

## k8s
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nacos
  namespace: default
spec:
  serviceName: nacos-hs
  replicas: 2
  template:
    metadata:
      labels:
        app: nacos
    spec:
      initContainers:
        - name: mysql2postgresql-jdbc-agent
          image: 221.214.10.82:81/mysql2postgresql-jdbc-agent/mysql2postgresql-jdbc-agent:1.0.0
          command: [ "/bin/sh" ]
          args: [ "-c", "cp -R /tmp/mysql2postgresql-jdbc-agent-1.0.0.jar /agent/" ]
          volumeMounts:
            - mountPath: /agent
              name: mysql2postgresql-jdbc-agent-volume
      containers:
        - name: nacos
          image: nacos/nacos-server:v2.1.0
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "512m"
          ports:
            - containerPort: 8848
              name: client
            - containerPort: 9848
              name: client-rpc
            - containerPort: 9849
              name: raft-rpc
            - containerPort: 7848
              name: old-raft-rpc
          env:
            - name: JAVA_OPT
              value: "-javaagent:/agent/mysql2postgresql-jdbc-agent-1.0.0.jar"
            - name: DB_URL_0
              value: "jdbc:postgresql://${MYSQL_SERVICE_HOST}:${MYSQL_SERVICE_PORT:5432}/${MYSQL_SERVICE_DB_NAME}?${MYSQL_SERVICE_DB_PARAM:currentSchema=public}"
            - name: MYSQL_SERVICE_HOST
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.db.host
            - name: MYSQL_SERVICE_DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.db.name
            - name: MYSQL_SERVICE_PORT
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.port
            - name: MYSQL_SERVICE_USER
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.user
            - name: MYSQL_SERVICE_PASSWORD
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.password
            - name: MYSQL_SERVICE_DB_PARAM
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.db.param
            - name: SPRING_DATASOURCE_PLATFORM
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.db.type
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.namespace
            - name: SERVICE_NAME
              value: "nacos-hs"
            - name: FUNCTION_MODE
              value: "config"
            - name: DOMAIN_NAME
              value: "cluster.local"
            - name: NACOS_APPLICATION_PORT
              value: "8848"
            - name: NACOS_SERVER_PORT
              value: "8848"
            - name: NACOS_REPLICAS
              value: "3"
            - name: MODE
              value: "cluster"
            - name: PREFER_HOST_MODE
              value: "hostname"
          volumeMounts:
            - name: mysql2postgresql-jdbc-agent-volume
              mountPath: /agent
      volumes:
        - name: mysql2postgresql-jdbc-agent-volume
          emptyDir: {}
  selector:
    matchLabels:
      app: nacos
```