# efk
efk收集日志
流程是这个样子，还有什么细节需要注意的么。
1. 创建namespace : logging ,后续所有资源在logging namespace下部署
2. StatefulSet 创建es数据库
3. 创建es对应的无头服务Service
4.创建 ServiceAccount fluentd 用于fluentd pod yaml的配置
5.ClusterRoleBinding 给创建的fluentd 绑定权限
6.daemonset创建 fluentd搜集 镜像： fluentd-kubernetes-daemonset
7.部署 kibana Pod
8.部署 kibana Service
9.部署ingress 访问 kibana
kind: StatefulSet
apiVersion: apps/v1
metadata:
  name: es-cluster
  namespace: logging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      initContainers:
        - name: fix-permissions
          image: busybox
          command:
            - sh
            - '-c'
            - chown -R 1000:1000 /usr/share/elasticsearch/data
          resources: {}
          volumeMounts:
            - name: data
              mountPath: /usr/share/elasticsearch/data
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: Always
          securityContext:
            privileged: true
        - name: increase-vm-max-map
          image: busybox
          command:
            - sysctl
            - '-w'
            - vm.max_map_count=262144
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: Always
          securityContext:
            privileged: true
        - name: increase-fd-ulimit
          image: busybox
          command:
            - sh
            - '-c'
            - ulimit -n 65536
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: Always
          securityContext:
            privileged: true
      containers:
        - name: elasticsearch
          image: docker.elastic.co/elasticsearch/elasticsearch:7.16.3
          ports:
            - name: rest
              containerPort: 9200
              protocol: TCP
            - name: inter
              containerPort: 9300
              protocol: TCP
          env:
            - name: cluster.name
              value: k8s-logs
            - name: node.name
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
            - name: discovery.seed_hosts
              value: es-cluster-0.elasticsearch
            - name: cluster.initial_master_nodes
              value: es-cluster-0
            - name: ES_JAVA_OPTS
              value: '-Xms512m -Xmx512m'
          resources:
            limits:
              cpu: '1'
            requests:
              cpu: 100m
          volumeMounts:
            - name: data
              mountPath: /usr/share/elasticsearch/data
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: IfNotPresent
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      securityContext: {}
      schedulerName: default-scheduler
  volumeClaimTemplates:
    - kind: PersistentVolumeClaim
      apiVersion: v1
      metadata:
        name: data
        creationTimestamp: null
        labels:
          app: elasticsearch
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 64Gi
        storageClassName: managed-premium-retain
        volumeMode: Filesystem
  serviceName: elasticsearch
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
  revisionHistoryLimit: 10
  
  
  
  
  ---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: logging

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: fluentd
  namespace: logging
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: logging
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
  labels:
    k8s-app: fluentd-logging
    version: v1
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-logging
      version: v1
  template:
    metadata:
      labels:
        k8s-app: fluentd-logging
        version: v1
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      tolerations:
        - key: CriticalAddonsOnly
          operator: Exists
        - effect: NoExecute
          operator: Exists
        - effect: NoSchedule
          operator: Exists
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.14-debian-elasticsearch7-1
        env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: elasticsearch.logging.svc.cluster.local
            - name: FLUENT_ELASTICSEARCH_PORT
              value: '9200'
            - name: FLUENT_ELASTICSEARCH_SCHEME
              value: http
            - name: FLUENTD_SYSTEMD_CONF
              value: disable
            - name: FLUENT_CONTAINER_TAIL_EXCLUDE_PATH
              value: /var/log/containers/fluent*
            - name: FLUENT_CONTAINER_TAIL_PARSER_TYPE
              value: /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
  
  
  
kind: Service
apiVersion: v1
metadata:
  name: elasticsearch
  namespace: logging
  uid: 743e46ca-b203-4889-9692-0450e081fd8c
  resourceVersion: '101662007'
  creationTimestamp: '2020-12-30T07:24:30Z'
  labels:
    app: elasticsearch
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: >
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"elasticsearch"},"name":"elasticsearch","namespace":"logging"},"spec":{"clusterIP":"None","ports":[{"name":"rest","port":9200},{"name":"inter-node","port":9300}],"selector":{"app":"elasticsearch"}}}
spec:
  ports:
    - name: rest
      protocol: TCP
      port: 9200
      targetPort: 9200
    - name: inter-node
      protocol: TCP
      port: 9300
      targetPort: 9300
  selector:
    app: elasticsearch
  clusterIP: None
  clusterIPs:
    - None
  type: ClusterIP
  sessionAffinity: None
status:
  loadBalancer: {}


kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: external-auth-oauth2
  namespace: logging
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: >
      {"apiVersion":"networking.k8s.io/v1beta1","kind":"Ingress","metadata":{"annotations":{"nginx.ingress.kubernetes.io/auth-signin":"https://$host/oauth2/start?rd=$escaped_request_uri","nginx.ingress.kubernetes.io/auth-url":"https://$host/oauth2/auth"},"name":"external-auth-oauth2","namespace":"logging"},"spec":{"rules":[{"host":"logging.siluzan.com","http":{"paths":[{"backend":{"serviceName":"kibana","servicePort":5601},"path":"/"}]}}]}}
    nginx.ingress.kubernetes.io/auth-signin: https://$host/oauth2/start?rd=$escaped_request_uri
    nginx.ingress.kubernetes.io/auth-url: https://$host/oauth2/auth
spec:
  rules:
    - host: logging.mysiluzan.com
      http:
        paths:
          - path: /
            pathType: ImplementationSpecific
            backend:
              service:
                name: kibana
                port:
                  number: 5601

  
  
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: oauth2-proxy
  namespace: logging
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: >
      {"apiVersion":"networking.k8s.io/v1beta1","kind":"Ingress","metadata":{"annotations":{},"name":"oauth2-proxy","namespace":"logging"},"spec":{"rules":[{"host":"logging.siluzan.com","http":{"paths":[{"backend":{"serviceName":"oauth2-proxy","servicePort":4180},"path":"/oauth2"}]}}],"tls":[{"hosts":["logging.siluzan.com"],"secretName":"oauth2-proxy-tls"}]}}
spec:
  tls:
    - hosts:
        - logging.mysiluzan.com
      secretName: tls-secert
  rules:
    - host: logging.mysiluzan.com
      http:
        paths:
          - path: /oauth2
            pathType: ImplementationSpecific
            backend:
              service:
                name: oauth2-proxy
                port:
                  number: 4180
  
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
  name: logging-ingress
  namespace: logging
spec:
  rules:
    - host: logging.mysiluzan.com
      http:
        paths:
          - backend:
              service:
                name: kibana
                port: 
                  number: 5601
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - logging.mysiluzan.com
      secretName: tls-secert

kind: Deployment
apiVersion: apps/v1
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
  annotations:
    deployment.kubernetes.io/revision: '4'
    kubectl.kubernetes.io/last-applied-configuration: >
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"labels":{"app":"kibana"},"name":"kibana","namespace":"logging"},"spec":{"replicas":1,"selector":{"matchLabels":{"app":"kibana"}},"template":{"metadata":{"labels":{"app":"kibana"}},"spec":{"containers":[{"env":[{"name":"ELASTICSEARCH_URL","value":"http://elasticsearch:9200"}],"image":"sammwp.azurecr.cn/kibana7:latest","name":"kibana","ports":[{"containerPort":5601}],"resources":{"limits":{"cpu":"1000m"},"requests":{"cpu":"100m"}}}],"nodeSelector":{"agentpool":"infra"},"tolerations":[{"key":"np","operator":"Exists"}]}}}}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
      annotations:
        kubectl.kubernetes.io/restartedAt: '2020-12-31T22:09:06-08:00'
    spec:
      containers:
        - name: kibana
          image: docker.elastic.co/kibana/kibana:7.6.2
          ports:
            - containerPort: 5601
              protocol: TCP
          env:
            - name: ELASTICSEARCH_URL
              value: http://elasticsearch:9200
          resources:
            limits:
              cpu: '1'
            requests:
              cpu: 100m
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: IfNotPresent
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      securityContext: {}
      schedulerName: default-scheduler
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600

kind: Service
apiVersion: v1
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: >
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"kibana"},"name":"kibana","namespace":"logging"},"spec":{"ports":[{"port":5601}],"selector":{"app":"kibana"}}}
spec:
  ports:
    - protocol: TCP
      port: 5601
      targetPort: 5601
  selector:
    app: kibana
  type: ClusterIP
  sessionAffinity: None
status:
  loadBalancer: {}

  
  
kind: Deployment
apiVersion: apps/v1
metadata:
  name: oauth2-proxy
  namespace: logging
  labels:
    k8s-app: oauth2-proxy
  annotations:
    deployment.kubernetes.io/revision: '12'
    kubectl.kubernetes.io/last-applied-configuration: >
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"labels":{"k8s-app":"oauth2-proxy"},"name":"oauth2-proxy","namespace":"logging"},"spec":{"replicas":1,"selector":{"matchLabels":{"k8s-app":"oauth2-proxy"}},"template":{"metadata":{"labels":{"k8s-app":"oauth2-proxy"}},"spec":{"containers":[{"args":["--provider=oidc","--client-id=logging","--client-secret=secret","--redirect-url=https://logging.siluzan.com/oauth2/callback","--oidc-issuer-url=https://sso.siluzan.com","--email-domain=*","--upstream=file:///dev/null","--http-address=0.0.0.0:4180","--pass-access-token=true","--user-id-claim=sub"],"env":[{"name":"OAUTH2_PROXY_COOKIE_SECRET","value":"UXZ1ZnBBL3Q0dWpmZjBzbFE1NmMxUT09"}],"image":"quay.io/oauth2-proxy/oauth2-proxy:latest","imagePullPolicy":"Always","name":"oauth2-proxy","ports":[{"containerPort":4180,"protocol":"TCP"}]}]}}}}
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: oauth2-proxy
  template:
    metadata:
      labels:
        k8s-app: oauth2-proxy
    spec:
      containers:
        - name: oauth2-proxy
          image: quay.io/oauth2-proxy/oauth2-proxy:latest
          args:
            - '--provider=oidc'
            - '--client-id=logging'
            - '--client-secret=secret'
            - '--redirect-url=https://logging.mysiluzan.com/oauth2/callback'
            - '--oidc-issuer-url=https://sso-ci.siluzan.com'
            - '--email-domain=*'
            - '--upstream=file:///dev/null'
            - '--http-address=0.0.0.0:4180'
            - '--pass-access-token=true'
            - '--user-id-claim=sub'
          ports:
            - containerPort: 4180
              protocol: TCP
          env:
            - name: OAUTH2_PROXY_COOKIE_SECRET
              value: UXZ1ZnBBL3Q0dWpmZjBzbFE1NmMxUT09
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: Always
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      securityContext: {}
      schedulerName: default-scheduler
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600

  
  
kind: Service
apiVersion: v1
metadata:
  name: oauth2-proxy
  namespace: logging
  labels:
    k8s-app: oauth2-proxy
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: >
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"k8s-app":"oauth2-proxy"},"name":"oauth2-proxy","namespace":"logging"},"spec":{"ports":[{"name":"http","port":4180,"protocol":"TCP","targetPort":4180}],"selector":{"k8s-app":"oauth2-proxy"}}}
spec:
  ports:
    - name: http
      protocol: TCP
      port: 4180
      targetPort: 4180
  selector:
    k8s-app: oauth2-proxy
  type: ClusterIP
  sessionAffinity: None
