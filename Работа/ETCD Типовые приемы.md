
Проверка состояния кластера на примере prod кубера в ГЕОПе. Запуск команд в любом **поде** etcd:
```bash
_ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key --write-out=table --endpoints=192.168.1.5:2379,192.168.1.4:2379,192.168.1.3:2379 endpoint status
```

```bash
_ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key --write-out=table --endpoints=192.168.1.5:2379,192.168.1.4:2379,192.168.1.3:2379 endpoint health
```
Сжатие бд etcd через консоль пода etcd на **каждой** ноде:
```bash
_ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key defrag
```
Пример "правильного" значения флага `--initial-cluster` в конфигурации `/etc/kubernetes/manifests/etcd.yml`:
```bash
--initial-cluster=s58-g100-c3=[https://192.168.7.4:2380,s58-g100-c1=https://192.168.7.5:2380,s58-g100-c2=https://192.168.7.3:2380](https://192.168.7.4:2380,s58-g100-c1=https/)
```
>[!tip]
Пояснение: В этом флаге должны быть перечислены все контрол-ноды.

#kubernetes #k8s 