---
title: Salut Minikube
content_type: tutorial
weight: 5
menu:
  main:
    title: "Începe"
    weight: 10
    post: >
      <p>Ești gata să-ți murdarești mâinile? Construiește un cluster Kubernetes simplu care rulează o aplicație de probă.</p>
card:
  name: tutorials
  weight: 10
---

<!-- overview -->
Acest tutorial îți arata cum să rulezi o aplicație de probă
în Kuberenetes folosind [Minikube](/docs/setup/learning-environment/minikube) și Katacoda.
Katacoda oferă un mediu gratuit de Kubernetes în browser.

{{< note >}}
Poți deasemenea să urmărești acest tutorial dacă ai instalat [Minikube local](/docs/tasks/tools/install-minikube/).
{{< /note >}}



## {{% heading "objectives" %}}


* Publică o aplicație de probă în Minikube.
* Rulează aplicația.
* Vezi logurile aplicației.



## {{% heading "prerequisites" %}}


Acest tutorial oferă o imagine a containerului care folosește NGINX pentru a răspunde tuturor solicitărilor.


<!-- lessoncontent -->

## Creează un cluster Minikube

1. Apasă **Launch Terminal**

    {{< kat-button >}}

{{< note >}}
    Daca ai instalat Minikube local, execută `minikube start`.
{{< /note >}}

2. Deschide dashboardul Kubernetes într-un browser:

    ```shell
    minikube dashboard
    ```

3. Exclusiv pentru mediul Katacoda: În partea de sus a panoului terminal, apasă pe semnul plus, apoi apasă **Select port to view on Host 1**.

4. Exclusiv pentru mediul Katacoda: Scrie `30000`, și apoi apasă **Display Port**.

## Creeaza un Deployment

Un Pod Kubernetes [*Pod*](/docs/concepts/workloads/pods/) este un grup de unul sau mai multe containere,
legate între ele în scopul administrării si comunicării în rețea. Podul din acest 
tutorial are un singur Container. Un [*Deployment*](/docs/concepts/workloads/controllers/deployment/)
Kubernetes verifica starea de sănătate a Podului si restartează Containerul Podului dacă se oprește. 
Deploymenturile sunt modul recomandat de a administra crearea și scalarea Podurilor.

1. Foloseste comanda `kubectl create` pentru a crea un Deployment care administreaza un Pod.
Podul rulează un Container pe baza imaginii Docker furnizate.

    ```shell
    kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.4
    ```

2. Vizualizează Deploymentul:

    ```shell
    kubectl get deployments
    ```

    Rezultatul e similar cu:

    ```
    NAME         READY   UP-TO-DATE   AVAILABLE   AGE
    hello-node   1/1     1            1           1m
    ```

3. Vizualizează Podul:

    ```shell
    kubectl get pods
    ```

    Rezultatul e similar cu:

    ```
    NAME                          READY     STATUS    RESTARTS   AGE
    hello-node-5f76cf6ccf-br9b5   1/1       Running   0          1m
    ```

4. Vizualizează evenimentele clusterului:

    ```shell
    kubectl get events
    ```

5. Vizualizează configurația `kubectl`:

    ```shell
    kubectl config view
    ```

{{< note >}}
Pentru mai multe informatii despre comenzile `kubectl`, vezi [prezentarea generală kubectl](/docs/reference/kubectl/overview/).
{{< /note >}}

## Creeaza un serviciu

În mod implicit, Podul este accesibil numai prin adresa IP internă din cadrul
clusterului Kubernetes. Pentru a face Containerul `helo-node` accesibil din afara
rețelei virtuale Kubernetes, trebuie sa expui Podul sub forma unui
[*Serviciu*](/docs/concepts/services-networking/service/) Kubernetes.

1. Expune Podul catre internet folosind comanda `kubectl expose`:

    ```shell
    kubectl expose deployment hello-node --type=LoadBalancer --port=8080
    ```


    Instrucțiunea `--type=LoadBalancer` indică faptul că vrei să-ți expui Serivicul
    în afara clusterului.

2. Vizualizează Serviciul pe care l-ai creeat:

    ```shell
    kubectl get services
    ```

    Rezultatul e similar cu:


    ```
    NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
    hello-node   LoadBalancer   10.108.144.78   <pending>     8080:30369/TCP   21s
    kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP          23m
    ```

    În cazul furnizorilor de servicii cloud care acceptă load balancers,
    o adresă IP externă ar fi furnizată pentru a accesa Serviciul. În Minikube,
    tipul `LoadBalancer` face Serviciul să fie accesibil prin intermediul comenzii `minikube service`.

3. Execută următoarea comandă:

    ```shell
    minikube service hello-node
    ```

4. Exclusiv pentru mediul Katacoda: Apasă pe semnul plus, apoi apasă **Select port to view on Host 1**.

5. Exclusiv pentru mediul Katacoda: Observă numarul de 5 cifre al portului afișat opus de `8080` în rezultatul serviciilor. Acest număr de port este generat arbitrar și poate fi diferit pentru tine. Scrie numarul în caseta text cu numărul de port, apoi apasă Display Port. Folosing exemplul de mai devreme, ai scrie `30369`

    Aceasta deschide o fereastră de browser care servește aplicația si afișeaza răspunsul aplicației.

## Activarea addons

Minikube are un set de {{< glossary_tooltip text="addons" term_id="addons" >}} încorporate care pot fi activate, dezactivate și deschise în mediul local Kubernetes

1. Listarea addonurilor acceptate în prezent:

    ```shell
    minikube addons list
    ```

    Rezultatul e similar cu:

    ```
    addon-manager: enabled
    dashboard: enabled
    default-storageclass: enabled
    efk: disabled
    freshpod: disabled
    gvisor: disabled
    helm-tiller: disabled
    ingress: disabled
    ingress-dns: disabled
    logviewer: disabled
    metrics-server: disabled
    nvidia-driver-installer: disabled
    nvidia-gpu-device-plugin: disabled
    registry: disabled
    registry-creds: disabled
    storage-provisioner: enabled
    storage-provisioner-gluster: disabled
    ```

2. Activează un addon, de exemplu, `metrics-server`:

    ```shell
    minikube addons enable metrics-server
    ```

    Rezultatul e similar cu:

    ```
    metrics-server was successfully enabled
    ```

3. Vizualizează Podul și Serviciul create:

    ```shell
    kubectl get pod,svc -n kube-system
    ```

    Rezultatul e similar cu:

    ```
    NAME                                        READY     STATUS    RESTARTS   AGE
    pod/coredns-5644d7b6d9-mh9ll                1/1       Running   0          34m
    pod/coredns-5644d7b6d9-pqd2t                1/1       Running   0          34m
    pod/metrics-server-67fb648c5                1/1       Running   0          26s
    pod/etcd-minikube                           1/1       Running   0          34m
    pod/influxdb-grafana-b29w8                  2/2       Running   0          26s
    pod/kube-addon-manager-minikube             1/1       Running   0          34m
    pod/kube-apiserver-minikube                 1/1       Running   0          34m
    pod/kube-controller-manager-minikube        1/1       Running   0          34m
    pod/kube-proxy-rnlps                        1/1       Running   0          34m
    pod/kube-scheduler-minikube                 1/1       Running   0          34m
    pod/storage-provisioner                     1/1       Running   0          34m

    NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
    service/metrics-server         ClusterIP   10.96.241.45    <none>        80/TCP              26s
    service/kube-dns               ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP       34m
    service/monitoring-grafana     NodePort    10.99.24.54     <none>        80:30002/TCP        26s
    service/monitoring-influxdb    ClusterIP   10.111.169.94   <none>        8083/TCP,8086/TCP   26s
    ```

4. Dezactivează `metrics-server`:

    ```shell
    minikube addons disable metrics-server
    ```

    Rezultatul e similar cu:

    ```
    metrics-server was successfully disabled
    ```

## Curățare

Acum poți curăța resursele anterior create din clusterul tau:

```shell
kubectl delete service hello-node
kubectl delete deployment hello-node
```

Opțional, opreste mașina virtuala (MV) a Minikube:

```shell
minikube stop
```

Opțional, sterge MV Minikube:

```shell
minikube delete
```



## {{% heading "whatsnext" %}}


* Învață mai multe despre [obiectele Deployment](/docs/concepts/workloads/controllers/deployment/).
* Învață mai multe despre [Publicare aplicațiilor](/docs/tasks/run-application/run-stateless-application-deployment/).
* Învață mai multe despre [obiectele Service](/docs/concepts/services-networking/service/).

