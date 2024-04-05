
### *Посмотреть историю деплойментов сервиса:*
```bash
helm history <name of service> -n <namespace>
```
### *Откатиться* на другие версию деплоймента:
```bash
helm rollback <name of service> <release number> -n <namespace>
```
### *Удаления* подов застрявших в Terminating:
```bash
for p in $(kubectl get pods -n longhorn-system | grep Terminating | awk '{print $1}'); do kubectl delete pod $p -n longhorn-system --force --grace-period=0; done
```
### *Как* найти все проблемные pod'ы, которые не в запущенном состоянии (т.е. не `Running`)?
```bash
kubectl get pods -A --field-selector=status.phase!=Running | grep -v Complete
```
> [!quote]
[https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/)
### *Получить* список узлов и количество pod'ов на них:
```bash
kubectl get po -o json --all-namespaces | \

jq '.items | group_by(.spec.nodeName) | map({"nodeName": .[0].spec.nodeName, "count": length}) | sort_by(.count)'
```
### *Вот* так с `kubectl top` можно получить pod'ы, которые потребляют максимальное количество процессора или памяти:
```bash
# cpu
kubectl top pods -A | sort --reverse --key 3 --numeric

# memory
kubectl top pods -A | sort --reverse --key 4 --numeric
```
### *Получить по каждому контейнеру каждого pod'а его limits и requests:**
```bash
kubectl get pods -n my-namespace -o=custom-columns='NAME:spec.containers[*].name,MEMREQ:spec.containers[*].resources.requests.memory,MEMLIM:spec.containers[*].resources.limits.memory,CPUREQ:spec.containers[*].resources.requests.cpu,CPULIM:spec.containers[*].resources.limits.cpu'
```
### *Удалить* под принудительно
```bash
kubectl delete pod <PODNAME> --grace-period=0 --force --namespace <NAMESPACE>
```
### *Посмотрим* на события в кластере за последний час:
```bash
kubectl get events -owide
```
### *Посмотрим* на работу сердца нашего кластера – etcd. Для этого обратимся к логам пода (или юнита):
```bash
kubectl logs -n kube-system etcd-cluster-m1 --follow --tail 1000
```
### *Вот так с `kubectl top` можно получить pod'ы, которые потребляют максимальное количество процессора или памяти:*
```bash
# cpu
kubectl top pods -A | sort --reverse --key 3 --numeric

# memory
kubectl top pods -A | sort --reverse --key 4 --numeric
```
### *Получить* внутренние IP-адреса узлов кластера:
```bash
kubectl get nodes -o json | \
jq -r '.items[].status.addresses[]? | select (.type == "InternalIP") | .address' | \
paste -sd "\n" -
```
### *Вывести* все сервисы и nodePort, которые они занимают:
```bash
kubectl get --all-namespaces svc -o json | \
jq -r '.items[] | [.metadata.name,([.spec.ports[].nodePort | tostring ] | join("|"))]| @tsv'
```
### *Как* скопировать все секреты из одного пространства имён в другое?
```bash
kubectl get secrets -o json --namespace namespace-old | \
jq '.items[].metadata.namespace = "namespace-new"' | \
kubectl create-f -
```
### Скалирование daemonset-ов в ноль:
```yml
spec:  
  nodeSelector:  
    non-existing: 'true'
```
### Привязка pod'а к определённой ноде:
```yml

#через nodeSelector:
spec:
  nodeSelector:
    kubernetes.io/hostname: s55-g95-c4

#через affinity
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
            - s55-g95-c1
            - s55-g95-c2
    preferredDuringSchedulingIgnoredDuringExecution:
    . - weight: 1
        preference:
          matchExpressions:
          . - key: another-node-label-key
              operator: In
              values:
                - another-node-label-value
```
Ingress nginx grpc:
```yml
annotations:
  nginx.ingress.kubernetes.io/backend-protocol: GRPC
  nginx.ingress.kubernetes.io/client-body-buffer-size: 1M
  nginx.ingress.kubernetes.io/proxy-body-size: 8m
  nginx.ingress.kubernetes.io/proxy-buffer-size: 16k
  nginx.ingress.kubernetes.io/proxy-max-temp-file-size: 1024m
  nginx.ingress.kubernetes.io/server-snippet: >-
	keepalive_time 1d;
	keepalive_timeout 600s;
	keepalive_requests 100000;
	proxy_next_upstream error timeout invalid_header http_502 http_503 http_504;
	proxy_next_upstream_tries 3;
	proxy_next_upstream_timeout 0;
	grpc_next_upstream error timeout http_502 http_503 http_504
	non_idempotent;
	grpc_next_upstream_tries 3;
	grpc_next_upstream_timeout 0;
	grpc_buffer_size 64000k;
	grpc_read_timeout 600s;
	grpc_send_timeout 600s;
	grpc_connect_timeout 600s;
	grpc_socket_keepalive on;
	output_buffers 4 3200k;
	grpc_set_header x-real-ip $remote_addr;
	grpc_set_header x-ray-id $request_id;
	client_body_timeout 600s;
	client_header_timeout 600s;
	client_max_body_size 0;
	proxy_connect_timeout 600s;
	proxy_read_timeout 600s;
	proxy_send_timeout 600s;
	proxy_request_buffering off;
	proxy_buffering off;
	proxy_socket_keepalive on;
	proxy_ignore_client_abort on;
	fastcgi_read_timeout 600s;
	send_timeout 600s;
	proxy_max_temp_file_size 0;
   nginx.ingress.kubernetes.io/service-upstream: 'true'
   nginx.ingress.kubernetes.io/ssl-redirect: 'true'
```


> [!quote]
> https://habr.com/ru/company/flant/blog/512762/


#k8s #kubernetes