
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
https://habr.com/ru/company/flant/blog/512762/


#k8s #kubernetes