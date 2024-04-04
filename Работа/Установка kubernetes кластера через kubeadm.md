>[!info]
Если при заходе на сервер не отображется история через history, то скорей всего вы загрузились не в bash а в dash исправить можно следующим образом
После чего нужно перелогиниться и все заработает

```bash
chsh -s /bin/bash
```
### *Необходимо выключить swap на серверах:*****
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
### ****Загрузить** модули ядра с помощью команды:

```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```
### *****Загружаем* модули:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```
### ****Установить** параметры ядра для Kubernetes с помощью команды:

```bash
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```
### ****Установка** CRIO runtime:
   
   Устанавливаем доп пакет для crio libseccomp и устанавливаем пакет для работы с apt через https и curl:
```bash
echo 'deb http://deb.debian.org/debian buster-backports main' > /etc/apt/sources.list.d/backports.list

apt update
apt install -y -t buster-backports libseccomp2 || apt update -y -t buster-backports libseccomp2
apt install curl ca-certificates apt-transport-https -y
```
 Устанавливаем сам crio.
```bash
OS=xUbuntu_20.04
VERSION=1.26

echo "deb [signed-by=/usr/share/keyrings/libcontainers-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list

echo "deb [signed-by=/usr/share/keyrings/libcontainers-crio-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list

mkdir -p /usr/share/keyrings

curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | gpg --dearmor -o /usr/share/keyrings/libcontainers-archive-keyring.gpg

curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/Release.key | gpg --dearmor -o /usr/share/keyrings/libcontainers-crio-archive-keyring.gpg

apt-get update
apt-get install cri-o cri-o-runc
```
включаем crio
```bash
sudo systemctl daemon-reload
sudo systemctl restart crio
sudo systemctl enable crio
systemctl status crio
```
### ****Установка** kubernetes:

```bash
sudo mkdir -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.26/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.26/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
Необходимо сделать инициализацию кластера на первой мастер ноду:
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --upload-certs --control-plane-endpoint=<IP или DNS адрес мастер ноды>
```
После инициализации первой ноды, необходимо сохранить строки подключения для ввода остальных нод в кластер:
Так выглядит строка для ввода мастер нод:
```bash
kubeadm join <IP или DNS мастера1>:6443 --token fatn03.qw1dqfp7sd1hrx3y --discovery-token-ca-cert-hash sha256:bab37552110b5b31706 --control-plane --certificate-key 8f7a98aabc0c5
```
А так выглядит строка для воркер нод:
```bash
kubeadm join srv-n-k8s-master01:6443 --token fatn03.qw1dqfp7sd1hrx3y --discovery-token-ca-cert-hash sha256:bab37552110b5b31706
```
 >[!warning]
 >Если токен был утерян или протух, то строку подключения можно сгенерировать следующим образом:
```bash
kubeadm init phase upload-certs --upload-certs
kubeadm token create --print-join-command <--выдаст строку для добавления воркер нод
```
>[!tip]
>Чтобы иметь возможность ввести мастер в кластер, к полученной выше строке необходимо добавить:
```
--control-plane --certificate-key <ключ полученный ппосле выполнения команды в первой строчке>
```
 Заходим на мастера и воркеры и выполняем команды join в кластер.
### **Установка Flannel:****
Скачиваем манифест:
```bash
wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Если при создании кластера мы выделяли сеть отличную от 10.244.0.0/16, то необходимо внести изменения в vim `kube-flannel.yml`

```bash
net-conf.json: |
{
"Network": "10.244.0.0/16",
"Backend": {
"Type": "vxlan"
	}
}
```
Далее применяем манифест: 
```
kubectl apply -f kube-flannel.yml
```
### ****правка** конфига coredns

После установки kubermetes возникла проболема, что под не может подключится по имени сервиса к другому поду. Исправил путем добавления строчки в configMan coreDNS:

```js
.:53 {

errors
health {
lameduck 5s
}

ready
rewrite name suffix .svc. .svc.cluster.local. # ----< Была добавлена эта строчка
kubernetes cluster.local in-addr.arpa ip6.arpa {
pods insecure
fallthrough in-addr.arpa ip6.arpa
ttl 30

}
prometheus :9153
forward . /etc/resolv.conf {
max_concurrent 1000
}

cache 30
loop
reload
loadbalance
}
```
### ****Установка** Longhorn:

На всех нодах кластера, где будут располагаться PVC Longhorn необходимо установить ряд пакетов.
1. open-iscsi:
```
Идем на https://github.com/open-iscsi/open-iscsi
```
Скачиваем последний релиз.
Загружаем архив на все нужные ноды и распоковываем. Далее переходим в директрию и выполняем команды:
```
./configure
make
make install
```
2. `sudo apt install nfs-common -y`
3. заходим на мастер ноду, устанавливаем пакет jq и запускаем скрипт проверки от лонгхорна:
```
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.5.3/scripts/environment_check.sh | bash

helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --version 1.5.3
```
10. настройка HaProxy:
1. Устанавливаем пакет на мастер ноды:
```
sudo apt install haproxy -y
```
#### **Настраиваем конфиг по пути /etc/haproxy/haproxy.cfg:****

```yml
global
	log /dev/log local0 info
	chroot /var/lib/haproxy
	pidfile /var/run/haproxy.pid
	maxconn 4000
	user haproxy
	group haproxy
	daemon

# turn on stats unix socket

stats socket /var/lib/haproxy/stats
#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------

defaults

	mode tcp
	log global
	option tcplog
	option dontlognull
	option http-server-close
	option forwardfor except 127.0.0.0/8
	option redispatch
	retries 3

	timeout http-request 20s
	timeout queue 1m
	timeout connect 20s
	timeout client 1m
	timeout server 1m
	timeout http-keep-alive 10s
	timeout check 10s
	maxconn 3000

frontend kube-apiserver
	bind *:6444
	mode tcp
	option tcplog
	default_backend kube-apiserver

frontend http
	bind *:80
	mode tcp
	option tcplog
	default_backend http

frontend https
	bind *:443
	mode tcp
	option tcplog
	default_backend https

frontend stats
	mode http
	bind *:32000
	stats enable
	stats uri /
	stats refresh 10s
	stats auth admin:BFTpasswordStat

backend http
	mode tcp
	option tcplog
	option tcp-check
	balance roundrobin
	default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
	server kube-http-1 172.24.19.249:9080 check
	server kube-http-2 172.24.19.250:9080 check
	server kube-http-3 172.24.19.251:9080 check

backend https
	mode tcp
	option tcplog
	option tcp-check
	balance roundrobin
	default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
	server kube-https-3 172.24.19.251:9443 check
	server kube-https-2 172.24.19.250:9443 check
	server kube-https-1 172.24.19.249:9443 check

backend kube-apiserver
	mode tcp
	option tcplog
	option tcp-check
	balance roundrobin
	default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
	server kube-apiserver-1 172.24.19.249:6443 check
	server kube-apiserver-2 172.24.19.250:6443 check
	server kube-apiserver-3 172.24.19.251:6443 check
```
#### Запускаем HAproxy: 
```bash
systemctl enable haproxysystemctl start haproxy
```
### ****настройка** keepalived:
```
apt install keepalive -y
```
#### **Конфиг Мастер Ноды:***
```yml
global_defs {
	router_id srv-n-k8s-master01
	vrrp_skip_check_adv_addr
	vrrp_garp_interval 0
	vrrp_gna_interval 0
}
vrrp_script chk_haproxy {
	script "killall -0 haproxy"
	interval 2
	weight 2
}
vrrp_instance haproxy-vip {
	state MASTER
	priority 101
	interface ens160
	smtp_alert
	virtual_router_id 202
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	virtual_ipaddress {
		172.24.20.20
	}
	track_script {
		chk_haproxy
	}
}
```
#### ****Конфиг** для Бекап Ноды
```yml
global_defs {
	router_id srv-n-k8s-master02
	vrrp_skip_check_adv_addr
	vrrp_garp_interval 0
	vrrp_gna_interval 0
}
vrrp_script chk_haproxy {
	script "killall -0 haproxy"
	interval 2
	weight 2
}
vrrp_instance haproxy-vip {
	state BACKUP
	priority 100
	interface ens160
	smtp_alert
	virtual_router_id 202
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	virtual_ipaddress {
		172.24.20.20
	}
	track_script {
		chk_haproxy
	}
}
```
### ****Перевыпускаем** когфиг Api-server:

Для того чтобы иметь возможность подключиться к кластеру через ip балансира его нужно прописать в сертификате api-server. Чтобы это сделать нужно выполнить следующее:
```bash
1. Копируем `kubeadm.yaml` на мастер ноду с помощью команды:
2. Необходимо добавить нужные адреса в раздел apiServer -> certSANs:

kubectl -n kube-system get configmap kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' > kubeadm.yaml

apiServer:
	certSANs:
	- 172.24.20.20
	- srv-n-k8s-master01
	- srv-n-k8s-master02
	- srv-n-k8s-master03
extraArgs:
    authorization-mode: Node,RBAC
    
1. Переносим старые сертфикаты, чтобы kubeadm смог создать новые:
mv /etc/kubernetes/pki/apiserver.{crt,key} ~

4. Генерируем новые сертификаты для api-server:
kubeadm init phase certs apiserver --config kubeadm.yaml

5. Перзапускаем поды api-server чтобы они подтянули новые сертификаты, проверяем что все работает штатно.
6. Добавлеям наш kubeadm.yaml в кластер: kubeadm init phase upload-config kubeadm --config kubeadm.yaml
```
### ****Генерация** Админ Конфига:
> [!tip] 
>Начиная с версии kebernetes > 1.24 перестали автоматически создаваться токены для сервис аккаунтов.

#### **Поэтому чтобы сгененировать Адним конфиг. Делаем следующее:****
1. Создаем Сервис Аккаунт:
```yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: devops
  namespace: default
```
2. Создаем секрет:
```yml
apiVersion: v1
kind: Secret
metadata:
  name: devops
annotations:
  kubernetes.io/service-account.name: devops
  type: kubernetes.io/service-account-token
```

#k8s #kubernetes #longhorn