---
reviewers:
- andreipetcu
title: Construirea de clustere mari
weight: 20
---

## Acceptanta

La {{< param "version" >}}, Kubernetes acceptă clustere cu până la 5000 de noduri. Mai precis, acceptăm configurații care îndeplinesc * toate * din următoarele criterii:

* Nu mai mult de 5000 de noduri
* Nu mai mult de 150000 de poduri
* Nu mai mult de 300000 containere totale
* Nu mai mult de 100 de poduri pe nod


## Instalare

Un cluster este un set de noduri (aparate fizice sau virtuale) care rulează agenți Kubernetes, gestionați de un „master”(planul de control la nivel de cluster).

În mod normal, numărul de noduri dintr-un cluster este controlat de valoarea `NUM_NODES` în fişierul `config-default.sh` din platforma specifică   (de exemplu, vezi [GCE's `config-default.sh`](http://releases.k8s.io/{{< param "githubbranch" >}}/cluster/gce/config-default.sh)).

Simpla schimbare a acestei valori la ceva foarte mare poate provoca eșecul scriptului de configurare pentru mulți furnizori de cloud. Un deployment GCE , de exemplu, va întampina probleme de cotă și nu va reuși să creeze clusterul.

Atunci când configurați un cluster Kubernetes mare, trebuie luate în considerare următoarele probleme.

### Probleme de cota

Pentru a evita să vă confruntați cu probleme de cota furnizorilor de cloud, atunci când creați un cluster cu mai multe noduri, luați în considerare:

* Măriți cota pentru lucruri precum CPU, IP-uri etc.
* În [GCE, de exemplu,](https://cloud.google.com/compute/docs/resource-quotas) veți dori să creșteți cota pentru:
* CPUs
* Instanțe VM
* Total disc persistent rezervat
* Adrese IP în utilizare
* Reguli de firewall
* Reguli de expediere
* Rutele
* Target pools
* Setarea scriptului de configurare, astfel încât să creeze noi VM-uri nod în loturi mai mici, cu așteptări între ele, pentru că unii furnizori de cloud limitează crearea de VM-uri.

### Stocare etcd

Pentru a îmbunătăți performanța clusterelor mari, stocăm evenimente într-o instanță etcd.

Când creați un cluster, scripturile de salt existente:

* pornesc și configurează o instanță suplimentară etcd
* configurează api-serverul pentru a-l utiliza pentru stocarea evenimentelor

### Dimensiunea master și componentele master

Pe GCE / Google Kubernetes Engine și AWS, `kube-up` configurează automat dimensiunea VM corespunzătoare pentru masterul dvs. în funcție de numărul de noduri
în clusterul dvs. La alți furnizori, va trebui să-l configurați manual. Pentru referință, dimensiunile pe care le folosim pe GCE sunt

* 1-5 noduri: n1-standard-1
* 6-10 noduri: n1-standard-2
* 11-100 noduri: n1-standard-4
* 101-250 noduri: n1-standard-8
* 251-500 noduri: n1-standard-16
* mai mult de 500 noduri: n1-standard-32

Și dimensiunile pe care le folosim pe AWS sunt

* 1-5 noduri: m3.medium
* 6-10 noduri: m3.large
* 11-100 noduri: m3.xlarge
* 101-250 noduri: m3.2xlarge
* 251-500 noduri: c4.4xlarge
* mai mult de 500 noduri: c4.8xlarge

{{< note >}}
Pe Google Kubernetes Engine, dimensiunea nodului principal se ajustează automat în funcție de dimensiunea clusterului. Pentru mai multe informații, consultați [acest articol](https://cloudplatform.googleblog.com/2017/11/Cutting-Cluster-Management-Fees-on-Google-Kubernetes-Engine.html).

Pe AWS, dimensiunile nodurilor principale sunt setate în prezent la momentul pornirii clusterului și nu se modifică, chiar dacă ulterior scalați clusterul în sus sau în jos, eliminând manual sau adăugând noduri sau utilizând un distribuitor automat de cluster.
{{< /note >}}

### Resurse Addon

Pentru a preveni scurgerile de memorie sau alte probleme de resurse din [cluster addons](https://releases.k8s.io/{{< param "githubbranch" >}}/cluster/addons) de la consumul tuturor resurselor disponibile pe un nod, Kubernetes stabilește limitele resurselor pe containerele addon pentru a limita resursele CPU și Memorie pe care le pot consuma (Vezi PR [#10653](http://pr.k8s.io/10653/files) and [#10778](http://pr.k8s.io/10778/files)).

De exemplu:

```yaml
  containers:
  - name: fluentd-cloud-logging
    image: k8s.gcr.io/fluentd-gcp:1.16
    resources:
      limits:
        cpu: 100m
        memory: 200Mi
```

Cu excepția Heapster, aceste limite sunt statice și se bazează pe datele pe care le-am colectat de la addon-uri care rulează pe clustere cu 4 noduri (Vezi [#10335](http://issue.k8s.io/10335#issuecomment-117861225)). Addon-urile consumă mult mai multe resurse atunci când se execută pe grupuri de clustere mari (see [#5880](http://issue.k8s.io/5880#issuecomment-113984085)). Deci, dacă un cluster mare este implementat fără a ajusta aceste valori, addon-urile pot fi ucise în mod continuu, deoarece continuă să lovească limitele.

Pentru a evita să vă confruntați cu probleme de resurse de addon a clusterului, atunci când creați un cluster cu mai multe noduri, luați în considerare următoarele:

* Scaleaza memoria și limitele procesorului pentru fiecare dintre următoarele addonuri, dacă sunt utilizate, pe măsură ce creșteți dimensiunea clusterului (există o replică a fiecărei manipulări a întregului cluster, astfel încât memoria și utilizarea procesorului tind să crească proporțional cu dimensiunea / încărcarea pe cluster) :
  * [InfluxDB și Grafana](http://releases.k8s.io/{{< param "githubbranch" >}}/cluster/addons/cluster-monitoring/influxdb/influxdb-grafana-controller.yaml)
  * [kubedns, dnsmasq, și sidecar](http://releases.k8s.io/{{< param "githubbranch" >}}/cluster/addons/dns/kube-dns/kube-dns.yaml.in)
  * [Kibana](http://releases.k8s.io/{{< param "githubbranch" >}}/cluster/addons/fluentd-elasticsearch/kibana-deployment.yaml)
* Scalați numărul de replici pentru următoarele addon-uri, dacă sunt utilizate, împreună cu dimensiunea clusterului (există mai multe replici ale fiecăreia, astfel încât replicile din ce în ce mai mari ar trebui să ajute la gestionarea încărcării crescute, dar, deoarece sarcina pe replică crește ușor, luați în considerare și creșterea limitelor procesorului / memoriei):
  * [elasticsearch](http://releases.k8s.io/{{< param "githubbranch" >}}/cluster/addons/fluentd-elasticsearch/es-statefulset.yaml)
* Creșteți ușor limitele de memorie și CPU pentru fiecare dintre următoarele addonuri, dacă sunt folosite, împreună cu dimensiunea clusterului (există o replică pe nod, dar utilizarea procesorului / memoriei crește ușor odată cu încărcarea / dimensiunea clusterului):
  * [FluentD cu ElasticSearch Plugin](http://releases.k8s.io/{{< param "githubbranch" >}}/cluster/addons/fluentd-elasticsearch/fluentd-es-ds.yaml)
  * [FluentD cu GCP Plugin](http://releases.k8s.io/{{< param "githubbranch" >}}/cluster/addons/fluentd-gcp/fluentd-gcp-ds.yaml)

Limitele resurselor Heapster sunt setate dinamic pe baza dimensiunii inițiale a clusterului(vezi [#16185](http://issue.k8s.io/16185)
și [#22940](http://issue.k8s.io/22940)). Dacă descoperi că Heapster rulează
fara resurse, ar trebui să ajustați formulele care calculează cererea de memorie heapster (consultați PR-urile pentru detalii).

Pentru indicații despre cum să detectați dacă addon-urile ating limitele resurselor, consultați secțiunea [Sectiunea de depanare a resurselor de calcul](/docs/concepts/configuration/manage-compute-resources-container/#troubleshooting).

În [viitor](http://issue.k8s.io/13048), preconizăm să stabilim toate limitele de resurse de addon-uri a clusterului în funcție de dimensiunea clusterului, și de a le regla dinamic dacă crești sau micșorezi clusterul tău.
Asteptam cu drag PR-uri care implementează aceste caracteristici.

### Permiterea de eroari mici a nodului la pornire

Din diferite motive (vezi [#18969](https://github.com/kubernetes/kubernetes/issues/18969) pentru mai multe detalii) ruland
`kube-up.sh` cu un foarte mare `NUM_NODES` poate eșua din cauza unui număr foarte mic de noduri care nu se ridica corect.
În prezent, aveți două opțiuni: reporniți clusterul (`kube-down.sh` și apoi `kube-up.sh` iar), sau inainte de
a rula `kube-up.sh` setați variabila de mediu `ALLOWED_NOTREADY_NODES` la orice valoare vă simțiți confortabil. Acest lucru va permite `kube-up.sh` sa functioneze cu mai putine `NUM_NODES` ridicate. Depinzând de
motivul eșecului, acele noduri suplimentare se pot alătura ulterior sau clusterul poate rămâne la o dimensiune de
`NUM_NODES - ALLOWED_NOTREADY_NODES`.
