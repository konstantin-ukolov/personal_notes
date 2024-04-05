>[!tip]
Работы проводить на мастер-нодах через de-консоль или через ssh, если доступно - не через lens.

Остановить службу kubelet

```bash
systemctl stop kubelet
```
Остановить запущенные контейнеры на узле
```bash
crictl rm -f --all
```
Проверить текущий срок действия сертификатов
```bash
kubeadm certs check-expiration
```
В файл `/etc/kubernetes/manifests/kube-controller-manager.yaml` дописать
```bash
- --cluster-signing-duration=87600h
или
- --experimental-cluster-signing-duration=87600h
```
в зависимости от версии k8s получится, например:
```yml
spec:
  containers:
  - command:
    - kube-controller-manager
    - --cluster-signing-duration=87600h
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --address=127.0.0.1
    - --leader-elect=true
```
Перевыпустить сертификаты
```bash
kubeadm certs renew all
```
Забекапить kubelet.conf
```
mv /etc/kubernetes/kubelet.conf /etc/kubernetes/kubelet.conf.old
```
 Остановить лишние API
```bash
systemctl stop docker.socket containerd
```
Сгенерировать новый kubelet.conf
```bash
kubeadm init phase kubeconfig kubelet
```
Вернуть ip адрес ноды в kubelet.conf берем из бэкапа kubelet.conf.old
Скопировать новый конфиг для kubectl
```
cp /etc/kubernetes/admin.conf /root/.kube/config
```
Запустить службы
```bash
systemctl restart docker containerd
systemctl start kubelet
````
Проверить состояние кластера
```
kubectl get node
```

#kubernetes #k8s 