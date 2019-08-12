#### HW 6. Networks
* Создал service, type=ClusterIP
* Проверил отстуствие его в ip addr
* В iptables создалось несколько цепочек для него
* Поправил ConfigMap kube-proxy, mode=ipvs
* Удалил под kube-proxy
* Сбросил правила iptables, kube-proxy заново добавил нужные убрав цепочки виртуальным ip
* Поставил в toolbox ipvsadm, ipset:
```
TCP  10.108.218.82:80 rr
  -> 172.17.0.4:8000              Masq    1      0          0
  -> 172.17.0.5:8000              Masq    1      0          0
  -> 172.17.0.6:8000              Masq    1      0          0
```

```
Name: KUBE-CLUSTER-IP
Type: hash:ip,port
Revision: 5
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 408
References: 2
Number of entries: 5
Members:
10.96.0.10,tcp:9153
10.108.218.82,tcp:80
10.96.0.10,udp:53
10.96.0.10,tcp:53
10.96.0.1,tcp:443
```

```
-A KUBE-SERVICES -m set --match-set KUBE-CLUSTER-IP dst,dst -j ACCEPT
```

##### MetalLB
* Добавил metallb через внешнюю ссылку, применил для него ConfigMap `config`
* Создал сервис типа LoadBalancer, добавил маршрут для 172.17.255.0/24 через minikube
* Проверил смену подов `curl 172.17.255.0 | grep HOST`

##### DNS через MetalLB
* Добавил [2 сервиса](coredns/dns-svc.yaml) для tcp и udp портов dns. Чтобы MetalLB понял что надо использовать один ip -
аннотация `metallb.universe.tf/allow-shared-ip`

##### Ingress
* Применил mandatory.yaml от nginx ingress
* Добавил LoadBalancer на контроллер nginx [nginx-lb](nginx-lb.yaml)
* `curl 172.17.255.1` вернул что-то от openresty, OK
* Добавил headless сервис для web: [web-svc-headless](web-svc-headless.yaml)
```
Name:	web-svc.default.svc.cluster.local
Address: 172.17.0.6
Name:	web-svc.default.svc.cluster.local
Address: 172.17.0.5
Name:	web-svc.default.svc.cluster.local
Address: 172.17.0.4
```
* Добавил Ingress на сервис web-svc на путь /web [web-ingress](web-ingress.yaml)
* `curl 172.17.255.2/web/index.html` возвращает нужные html из разных подов

##### Dashboard
Добавил 2 Ingress [dashboard-ingress](dashboard/dashboard-ingress.yaml).
Один на путь /dashboard, перенаправление в корень. 
Второй - для ресурсов `static|api|assets`, перенаправление по соответствующим путям.

Также нужно добавить аннотацию `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`, потому что
dashboard работает только через https.

##### Canary
Немного поменял под из первого задания, чтобы он писал свою версию [hello-deploy](canary/hello-deploy.yaml)
`echo "hello, im stable" >> /app/index.html`

Еще один Deploy симулирует новую версию [hello-canary-deploy](canary/hello-canary-deploy.yaml) 
`echo "hello, im canary" >> /app/index.html`

Сделал для них по headless-сервису.

Добавил 2 Ingress'а для сервисов [hello-ingress](canary/hello-ingress.yaml).
Первый - для stable, второй - для canary версии. Для удобной проверки перенаправление сделал только по хэдеру
`nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"`

Пришлось использовать хост hello.com, потому что без них контроллер не могу слить второй ингресс на тот же путь.

Проверка:
```
curl -H "Host: hello.com" http://172.17.255.2/hello
hello, im stable
```
```
curl -H "X-Canary: always" -H "Host: hello.com" http://172.17.255.2/hello
hello, im canary
```
