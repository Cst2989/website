---
reviewers:
- andreipetcu
title: Rularea în mai multe zone
weight: 10
content_type: concept
---

<!-- overview -->

Această pagină descrie cum să executați un cluster în mai multe zone.



<!-- body -->

## Introducere

Kubernetes 1.2 adaugă suport pentru rularea unui singur cluster în mai multe zone de eșec
(GCE le numește simplu „zone”, AWS le numește „zone de disponibilitate”, aici vom face referire la ele drept „zone”).
Aceasta este o versiune ușoară a unei funcții mai largi de Federație Cluster (menționată anterior afectuos cu porecla ["Ubernetes"](https://github.com/kubernetes/community/blob/{{< param "githubbranch" >}}/contributors/design-proposals/multicluster/federation.md)).
Federație Full Cluster  permite combinarea separată
de clustere Kubernetes care rulează în diferite regiuni sau furnizori de cloud
(sau centre de date locale). Cu toate acestea, mulți
utilizatorii doresc pur și simplu să ruleze un cluster Kubernetes mai disponibil în mai multe zone
a furnizorului lor de cloud, și asta este ceea ce permite suportul multizone din 1.2

Asistența multizone este limitată în mod deliberat: un singur cluster Kubernetes poate rula
în mai multe zone, dar numai în aceeași regiune (și furnizor de cloud).  Numai
GCE și AWS sunt acceptate în mod automat (deși este ușor de
adăugat suport similar pentru alți provideri de cloud sau chiar metal gol, prin simpla adaugare
de etichetele corespunzătoare la noduri și volume).


## Funcționalitate

Când nodurile sunt pornite, kubeletul le adaugă automat etichete cu
informații despre zonă

Kubernetes va răspândi automat podurile într-un controler de replicare
sau servicii peste noduri dintr-un cluster cu o singură zonă (pentru a reduce impactul
de eșecuri.)  Cu clustere cu mai multe zone, acest comportament de răspândire este
extins pe zone (pentru a reduce impactul defecțiunilor zonei.)  (Aceasta este
realizat prin  `SelectorSpreadPriority`).  Acesta este un efort de cea mai buna
plasare, și deci dacă zonele din clusterul dvs. sunt eterogene
(de exemplu. numere diferite de noduri, diferite tipuri de noduri sau
cerințe diferite de resurse), acest lucru ar putea preveni
răspândirea perfecta a podurile pe zone. Dacă doriți, puteți utiliza
zone omogene (același număr și tipuri de noduri) pentru reducerea
probabilitatea de răspândire inegală.

Când sunt create volume persistente, `PersistentVolumeLabel`
le adaugă automat etichete de zonă.  Planificatorul (prin intermediul
`VolumeZonePredicate` predicat) se va asigura apoi că podurile care pretind
volumul dat sunt plasate doar în aceeași zonă cu acel volum, ca volumele
nu poate fi atașate în alte zone.

## Limitări

Există câteva limitări importante ale suportului multizone:

* Presupunem că diferitele zone sunt situate aproape una de cealaltă în zona
de rețea, deci nu efectuăm nicio rutare conștientă de zonă.  În special, traficul
care trece prin servicii ar putea traversa zone (chiar dacă unele poduri care susțin acel serviciu
există în aceeași zonă cu clientul), iar acest lucru poate invoca latență și costuri suplimentare.

* Volumul-afinitate de zonă va funcționa doar cu `PersistentVolume`, și nu va
funcționa dacă specificați direct un volum EBS în spec (de exemplu).

* Culsterele nu se pot întinde pe mai mulți provideri de cloud sau regiuni (această funcționalitate va necesita complet
sprijin de la federație).

* Deși nodurile dvs. sunt în mai multe zone, kube-up  construiește în prezent
implicit un singur nod master.  În timp ce serviciile sunt extrem de
disponibile și pot tolera pierderea unei zone, planul de control este
situat într-o singură zonă.  Utilizatorii care doresc un planul de control extrem de disponibil
ar trebui să urmeze instrucțiunile de [Valabilitate ridicată](/docs/admin/high-availability).

### Limitări de volum
Următoarele limitări sunt adresate cu [legarea volumului conștientă de topologie](/docs/concepts/storage/storage-classes/#volume-binding-mode).

* StatefulSet răspândirea zonei de volum la utilizarea provizionării dinamice nu este în prezent compatibilă cu
  politici de afinitate sau anti-afinitate.

* Dacă numele StatefulSet conține liniuțe ("-"), răspândirea zonei de volum
  s-ar putea să nu asigure o distribuție uniformă a stocării pe zone.

* Când se specifică mai multe PVC-uri în un Deployment sau Pod spec,  StorageClass
  trebuie să fie configurat pentru o singură zonă specifică,sau PV-urile trebuie să fie
  prevazut static intr-o anumita zona. O altă soluție este utilizarea unui
  StatefulSet, care va asigura că toate volumele pentru o replică sunt
  situate în aceeași zonă.

## Parcurgere

Acum vom parcurge prin configurarea și utilizarea unei zone multiple
cluster atât pe GCE cât și pe AWS. Pentru a face acest lucru, creați un grup complet
(specificând `MULTIZONE=true`), apoi adăugați noduri în zone suplimentare
rulând `kube-up` din nou (specificând `KUBE_USE_EXISTING_MASTER=true`).

### Bringing up your cluster

Creați clusterul ca normal, dar treceți MULTIZONE pentru a spune clusterului să gestioneze mai multe zone; crearea nodurilor în us-central1-a.

GCE:

```shell
curl -sS https://get.k8s.io | MULTIZONE=true KUBERNETES_PROVIDER=gce KUBE_GCE_ZONE=us-central1-a NUM_NODES=3 bash
```

AWS:

```shell
curl -sS https://get.k8s.io | MULTIZONE=true KUBERNETES_PROVIDER=aws KUBE_AWS_ZONE=us-west-2a NUM_NODES=3 bash
```

Acest pas creează un cluster la fel de normal, care rulează încă într-o singură zonă
(dar `MULTIZONE=true` a activat capabilități cu mai multe zone).

### Nodurile sunt etichetate

Vizualizați nodurile; puteți vedea că sunt etichetate cu informații despre zonă.
Toate sunt în `us-central1-a` (GCE) sau `us-west-2a` (AWS) pana acum.
Etichetele sunt `failure-domain.beta.kubernetes.io/region` pentru regiune,
și `failure-domain.beta.kubernetes.io/zone` pentru zone:

```shell
kubectl get nodes --show-labels
```

Output-ul este similar cu acesta:

```shell
NAME                     STATUS                     ROLES    AGE   VERSION          LABELS
kubernetes-master        Ready,SchedulingDisabled   <none>   6m    v1.13.0          beta.kubernetes.io/instance-type=n1-standard-1,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-a,kubernetes.io/hostname=kubernetes-master
kubernetes-minion-87j9   Ready                      <none>   6m    v1.13.0          beta.kubernetes.io/instance-type=n1-standard-2,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-a,kubernetes.io/hostname=kubernetes-minion-87j9
kubernetes-minion-9vlv   Ready                      <none>   6m    v1.13.0          beta.kubernetes.io/instance-type=n1-standard-2,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-a,kubernetes.io/hostname=kubernetes-minion-9vlv
kubernetes-minion-a12q   Ready                      <none>   6m    v1.13.0          beta.kubernetes.io/instance-type=n1-standard-2,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-a,kubernetes.io/hostname=kubernetes-minion-a12q
```

### Adăugați mai multe noduri într-o a doua zonă

Să adăugăm un alt set de noduri la clusterul existent, reutilizând
maestru existent, care rulează într-o zonă diferită (us-central1-b sau us-west-2b).
Rulăm kube-up din nou, dar precizând `KUBE_USE_EXISTING_MASTER=true`
kube-up nu va crea un nou master, dar va reutiliza unul care a fost anterior
creat.

GCE:

```shell
KUBE_USE_EXISTING_MASTER=true MULTIZONE=true KUBERNETES_PROVIDER=gce KUBE_GCE_ZONE=us-central1-b NUM_NODES=3 kubernetes/cluster/kube-up.sh
```

Pe AWS trebuie să specificăm și CIDR-ul pentru rețeaua
Subnet, împreună cu adresa IP internă principală:

```shell
KUBE_USE_EXISTING_MASTER=true MULTIZONE=true KUBERNETES_PROVIDER=aws KUBE_AWS_ZONE=us-west-2b NUM_NODES=3 KUBE_SUBNET_CIDR=172.20.1.0/24 MASTER_INTERNAL_IP=172.20.0.9 kubernetes/cluster/kube-up.sh
```


Vizualizați din nou nodurile; Încă 3 noduri ar fi trebuit să fie lansate și să fie etichetate
în us-central1-b:

```shell
kubectl get nodes --show-labels
```

Output-ul este similar cu acesta:

```shell
NAME                     STATUS                     ROLES    AGE   VERSION           LABELS
kubernetes-master        Ready,SchedulingDisabled   <none>   16m   v1.13.0           beta.kubernetes.io/instance-type=n1-standard-1,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-a,kubernetes.io/hostname=kubernetes-master
kubernetes-minion-281d   Ready                      <none>   2m    v1.13.0           beta.kubernetes.io/instance-type=n1-standard-2,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-b,kubernetes.io/hostname=kubernetes-minion-281d
kubernetes-minion-87j9   Ready                      <none>   16m   v1.13.0           beta.kubernetes.io/instance-type=n1-standard-2,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-a,kubernetes.io/hostname=kubernetes-minion-87j9
kubernetes-minion-9vlv   Ready                      <none>   16m   v1.13.0           beta.kubernetes.io/instance-type=n1-standard-2,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-a,kubernetes.io/hostname=kubernetes-minion-9vlv
kubernetes-minion-a12q   Ready                      <none>   17m   v1.13.0           beta.kubernetes.io/instance-type=n1-standard-2,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-a,kubernetes.io/hostname=kubernetes-minion-a12q
kubernetes-minion-pp2f   Ready                      <none>   2m    v1.13.0           beta.kubernetes.io/instance-type=n1-standard-2,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-b,kubernetes.io/hostname=kubernetes-minion-pp2f
kubernetes-minion-wf8i   Ready                      <none>   2m    v1.13.0           beta.kubernetes.io/instance-type=n1-standard-2,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-b,kubernetes.io/hostname=kubernetes-minion-wf8i
```

### Afinitatea de volum

Creați un volum utilizând crearea dinamică a volumului(doar PersistentVolumes sunt acceptate pentru afinitatea zonelor):

```bash
kubectl apply -f - <<EOF
{
  "apiVersion": "v1",
  "kind": "PersistentVolumeClaim",
  "metadata": {
    "name": "claim1",
    "annotations": {
        "volume.alpha.kubernetes.io/storage-class": "foo"
    }
  },
  "spec": {
    "accessModes": [
      "ReadWriteOnce"
    ],
    "resources": {
      "requests": {
        "storage": "5Gi"
      }
    }
  }
}
EOF
```

{{< note >}}
Pentru versiunea 1.3+ Kubernetes va distribui claim-uri PV dinamice peste
zonele configurate. Pentru versiunea 1.2, PV-urile dinamice au fost
întotdeauna create în zona masterului clusterului
(aici us-central1-a / us-west-2a); acest issue
([#23330](https://github.com/kubernetes/kubernetes/issues/23330))
a fost abordat în versiunea 1.3+.
{{< /note >}}

Acum să validăm faptul că Kubernetes a etichetat automat zona și regiunea în care a fost creat PV-ul.

```shell
kubectl get pv --show-labels
```

Output-ul este similar cu acesta:

```shell
NAME           CAPACITY   ACCESSMODES   RECLAIM POLICY   STATUS    CLAIM            STORAGECLASS    REASON    AGE       LABELS
pv-gce-mj4gm   5Gi        RWO           Retain           Bound     default/claim1   manual                    46s       failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-a
```

Deci, acum vom crea un pod care folosește revendicarea volumului persistent.
Deoarece volumele GBS PD / AWS EBS nu pot fi atașate în zone,
asta înseamnă că acest pod poate fi creat doar în aceeași zonă cu volumul:

```yaml
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: claim1
EOF
```

Rețineți că podul a fost creat automat în aceeași zonă cu volumul ca
atașamentele între zone nu sunt în general permise de furnizorii de cloud:

```shell
kubectl describe pod mypod | grep Node
```

```shell
Node:        kubernetes-minion-9vlv/10.240.0.5
```

Și verificați etichetele nodurilor:

```shell
kubectl get node kubernetes-minion-9vlv --show-labels
```

```shell
NAME                     STATUS    AGE    VERSION          LABELS
kubernetes-minion-9vlv   Ready     22m    v1.6.0+fff5156   beta.kubernetes.io/instance-type=n1-standard-2,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-a,kubernetes.io/hostname=kubernetes-minion-9vlv
```

### Podurile sunt răspândite pe zone

Podurile dintr-un controler sau serviciu de replicare sunt răspândite automat
de-a lungul zonelor.  În primul rând, să lansăm mai multe noduri într-o a treia zonă:

GCE:

```shell
KUBE_USE_EXISTING_MASTER=true MULTIZONE=true KUBERNETES_PROVIDER=gce KUBE_GCE_ZONE=us-central1-f NUM_NODES=3 kubernetes/cluster/kube-up.sh
```

AWS:

```shell
KUBE_USE_EXISTING_MASTER=true MULTIZONE=true KUBERNETES_PROVIDER=aws KUBE_AWS_ZONE=us-west-2c NUM_NODES=3 KUBE_SUBNET_CIDR=172.20.2.0/24 MASTER_INTERNAL_IP=172.20.0.9 kubernetes/cluster/kube-up.sh
```

Verificați că acum aveți noduri în 3 zone:

```shell
kubectl get nodes --show-labels
```

Creați exemplul de carte de oaspeți, care include un RC de dimensiunea 3, care rulează o aplicație web simplă:

```shell
find kubernetes/examples/guestbook-go/ -name '*.json' | xargs -I {} kubectl apply -f {}
```

Podurile ar trebui să fie distribuite pe toate cele 3 zone:

```shell
kubectl describe pod -l app=guestbook | grep Node
```

```shell
Node:        kubernetes-minion-9vlv/10.240.0.5
Node:        kubernetes-minion-281d/10.240.0.8
Node:        kubernetes-minion-olsh/10.240.0.11
```

```shell
kubectl get node kubernetes-minion-9vlv kubernetes-minion-281d kubernetes-minion-olsh --show-labels
```

```shell
NAME                     STATUS    ROLES    AGE    VERSION          LABELS
kubernetes-minion-9vlv   Ready     <none>   34m    v1.13.0          beta.kubernetes.io/instance-type=n1-standard-2,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-a,kubernetes.io/hostname=kubernetes-minion-9vlv
kubernetes-minion-281d   Ready     <none>   20m    v1.13.0          beta.kubernetes.io/instance-type=n1-standard-2,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-b,kubernetes.io/hostname=kubernetes-minion-281d
kubernetes-minion-olsh   Ready     <none>   3m     v1.13.0          beta.kubernetes.io/instance-type=n1-standard-2,failure-domain.beta.kubernetes.io/region=us-central1,failure-domain.beta.kubernetes.io/zone=us-central1-f,kubernetes.io/hostname=kubernetes-minion-olsh
```


Load-balancers se întind în toate zonele dintr-un cluster; exemplu de carte de oaspeți
include un exemplu de serviciu de load balance:

```shell
kubectl describe service guestbook | grep LoadBalancer.Ingress
```

Output-ul este similar cu acesta:

```shell
LoadBalancer Ingress:   130.211.126.21
```

Setați IP-ul de mai sus:

```shell
export IP=130.211.126.21
```

Explorați cu curl IP-ul:

```shell
curl -s http://${IP}:3000/env | grep HOSTNAME
```

Output-ul este similar cu acesta:

```shell
  "HOSTNAME": "guestbook-44sep",
```

Din nou, explorați de mai multe ori:

```shell
(for i in `seq 20`; do curl -s http://${IP}:3000/env | grep HOSTNAME; done)  | sort | uniq
```

Output-ul este similar cu acesta:

```shell
  "HOSTNAME": "guestbook-44sep",
  "HOSTNAME": "guestbook-hum5n",
  "HOSTNAME": "guestbook-ppm40",
```

Load Balancer-ul vizează corect toate podurile, chiar dacă se află în mai multe zone.

### Oprirea clusterului

Când ați terminat, curățați:

GCE:

```shell
KUBERNETES_PROVIDER=gce KUBE_USE_EXISTING_MASTER=true KUBE_GCE_ZONE=us-central1-f kubernetes/cluster/kube-down.sh
KUBERNETES_PROVIDER=gce KUBE_USE_EXISTING_MASTER=true KUBE_GCE_ZONE=us-central1-b kubernetes/cluster/kube-down.sh
KUBERNETES_PROVIDER=gce KUBE_GCE_ZONE=us-central1-a kubernetes/cluster/kube-down.sh
```

AWS:

```shell
KUBERNETES_PROVIDER=aws KUBE_USE_EXISTING_MASTER=true KUBE_AWS_ZONE=us-west-2c kubernetes/cluster/kube-down.sh
KUBERNETES_PROVIDER=aws KUBE_USE_EXISTING_MASTER=true KUBE_AWS_ZONE=us-west-2b kubernetes/cluster/kube-down.sh
KUBERNETES_PROVIDER=aws KUBE_AWS_ZONE=us-west-2a kubernetes/cluster/kube-down.sh
```


