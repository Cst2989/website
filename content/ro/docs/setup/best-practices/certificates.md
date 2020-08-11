---
title: Certificate și cerințe PKI
reviewers:
- andreipetcu
content_type: concept
weight: 40
---

<!-- overview -->

Kubernetes necesită certificate PKI pentru autentificare prin TLS.
Dacă instalați Kubernetes cu [kubeadm](/docs/reference/setup-tools/kubeadm/kubeadm/), certificatele pe care le necesită clusterul dvs. sunt generate automat.
De asemenea, puteți genera propriile dvs. certificate - de exemplu, pentru a vă păstra cheile private mai sigure, fără a le stoca pe serverul API.
Această pagină explică certificatele de care are nevoie clusterul dvs.



<!-- body -->

## Modul în care certificatele sunt utilizate de clusterul dvs.

Kubernetes necesită PKI pentru următoarele operații:

* Certificate de client pentru kubelet pentru autentificarea cu serverul API
* Certificat de server pentru serverul API
* Certificate de client pentru administratorii clusterului pentru autentificarea cu serverul API
* Certificate de client pentru serverul API pentru a comunica cu kubelets
* Certificat de client pentru serverul API pentru a comunica cu etcd
* Certificat de client / kubeconfig pentru managerul controlorului să comunice cu serverul API
* Certificat de client / kubeconfig pentru care programatorul comunica cu serverul API.
* Certificate de client și server pentru [front-proxy](/docs/tasks/extend-kubernetes/configure-aggregation-layer/)

{{< note >}}
`front-proxy` certificatele sunt necesare numai dacă executați kube-proxy să suporte [un server API de extensie](/docs/tasks/extend-kubernetes/setup-extension-api-server/).
{{< /note >}}

etcd implementează, de asemenea, TLS reciproce pentru autentificarea clienților și colegilor.

## Unde sunt stocate certificatele

Dacă instalați Kubernetes cu kubeadm, certificatele sunt stocate în `/etc/kubernetes/pki`. Toate căile din această documentație sunt relative la acel director.

## Configurați certificatele manual

Dacă nu doriți ca kubeadm să genereze certificatele necesare, le puteți crea în oricare dintre următoarele moduri.

### CA rădăcină unică

Puteți crea o singură CA rădăcină, controlată de un administrator. Această CA-rădăcină poate crea apoi mai multe CA intermediare și poate delega toate creațiile ulterioare către Kubernetes.

CA obligatorii:

| cale                   | Mod implicit CN                | Descriere                      |
|------------------------|---------------------------|----------------------------------|
| ca.crt,key             | kubernetes-ca             | CA general Kubernetes           |
| etcd/ca.crt,key        | etcd-ca                   | Pentru toate funcțiile legate de etcd   |
| front-proxy-ca.crt,key | kubernetes-front-proxy-ca | Pentru [front-end proxy](/docs/tasks/extend-kubernetes/configure-aggregation-layer/) |

Înafara de CA-urile de mai sus, este necesar, de asemenea, să obțineți o pereche de chei publice / private pentru gestionarea contului de serviciu, `sa.key` și `sa.pub`.

### Toate certificatele

Dacă nu doriți să copiați cheile private CA în clusterul dvs., puteți genera singur toate certificatele.

Certificate necesare:

| Mod implicit CN                    | Părinte CA                 | O (în Subiect) | kind                                   | hosts (SAN)                                 |
|-------------------------------|---------------------------|----------------|----------------------------------------|---------------------------------------------|
| kube-etcd                     | etcd-ca                   |                | server, client                         | `localhost`, `127.0.0.1`                        |
| kube-etcd-peer                | etcd-ca                   |                | server, client                         | `<hostname>`, `<Host_IP>`, `localhost`, `127.0.0.1` |
| kube-etcd-healthcheck-client  | etcd-ca                   |                | client                                 |                                             |
| kube-apiserver-etcd-client    | etcd-ca                   | system:masters | client                                 |                                             |
| kube-apiserver                | kubernetes-ca             |                | server                                 | `<hostname>`, `<Host_IP>`, `<advertise_IP>`, `[1]` |
| kube-apiserver-kubelet-client | kubernetes-ca             | system:masters | client                                 |                                             |
| front-proxy-client            | kubernetes-front-proxy-ca |                | client                                 |                                             |

[1]: orice alt nume IP sau DNS cu care vă contactați clusterul(folosit de  [kubeadm](/docs/reference/setup-tools/kubeadm/kubeadm/)
load balancer IP stabil și / sau nume DNS, `kubernetes`, `kubernetes.default`, `kubernetes.default.svc`,
`kubernetes.default.svc.cluster`, `kubernetes.default.svc.cluster.local`)

unde `kind` se mapeaza la una sau mai multe [x509 key usage](https://godoc.org/k8s.io/api/certificates/v1beta1#KeyUsage) tipuri:

| kind   | Utilizarea cheilor                                                                |
|--------|---------------------------------------------------------------------------------|
| server | semnătura digitală, codul cheilor, autentificarea serverului                                |
| client | semnătura digitală, codul cheilor, autentificarea clientului                          |

{{< note >}}
Gazdele / SAN enumerate mai sus sunt cele recomandate pentru obținerea unui cluster de lucru; dacă este necesară o configurație specifică, este posibil să adăugați SAN-uri suplimentare la toate certificatele de server.
{{< /note >}}

{{< note >}}
Doar pentru utilizatorii kubeadm:

* Scenariul în care copiați certificatele CA de cluster fără chei private este denumit CA extern în documentația kubeadm.
* Dacă comparați lista de mai sus cu un PKI generat de kubeadm, vă rugăm să realizați că certificatele `kube-etcd`, `kube-etcd-peer` și `kube-etcd-healthcheck-client` nu sunt generate în cazul de etcd extern.

{{< /note >}}

### Căile certificatelor

Certificatele trebuie plasate într-o cale recomandată (așa cum este folosit de [kubeadm](/docs/reference/setup-tools/kubeadm/kubeadm/)).
Căile trebuie specificate folosind argumentul dat indiferent de locație.

| Mod implicit CN                   | calea cheie recomandată        | cale cert recomandată     | comanda        | argumentul cheie          |  argumentul cert                   |
|------------------------------|------------------------------|-----------------------------|----------------|------------------------------|-------------------------------------------|
| etcd-ca                      |     etcd/ca.key                         | etcd/ca.crt                 | kube-apiserver |                              | --etcd-cafile                             |
| kube-apiserver-etcd-client   | apiserver-etcd-client.key    | apiserver-etcd-client.crt   | kube-apiserver | --etcd-keyfile               | --etcd-certfile                           |
| kubernetes-ca                |    ca.key                          | ca.crt                      | kube-apiserver |                              | --client-ca-file                          |
| kubernetes-ca                |    ca.key                          | ca.crt                      | kube-controller-manager | --cluster-signing-key-file      | --client-ca-file, --root-ca-file, --cluster-signing-cert-file  |
| kube-apiserver               | apiserver.key                | apiserver.crt               | kube-apiserver | --tls-private-key-file       | --tls-cert-file                           |
| kube-apiserver-kubelet-client|     apiserver-kubelet-client.key                         | apiserver-kubelet-client.crt| kube-apiserver | --kubelet-client-key | --kubelet-client-certificate              |
| front-proxy-ca               |     front-proxy-ca.key                         | front-proxy-ca.crt          | kube-apiserver |                              | --requestheader-client-ca-file            |
| front-proxy-ca               |     front-proxy-ca.key                         | front-proxy-ca.crt          | kube-controller-manager |                              | --requestheader-client-ca-file |
| front-proxy-client           | front-proxy-client.key       | front-proxy-client.crt      | kube-apiserver | --proxy-client-key-file      | --proxy-client-cert-file                  |
| etcd-ca                      |         etcd/ca.key                     | etcd/ca.crt                 | etcd           |                              | --trusted-ca-file, --peer-trusted-ca-file |
| kube-etcd                    | etcd/server.key              | etcd/server.crt             | etcd           | --key-file                   | --cert-file                               |
| kube-etcd-peer               | etcd/peer.key                | etcd/peer.crt               | etcd           | --peer-key-file              | --peer-cert-file                          |
| etcd-ca                      |                              | etcd/ca.crt                 | etcdctl    |                              | --cacert                                  |
| kube-etcd-healthcheck-client | etcd/healthcheck-client.key  | etcd/healthcheck-client.crt | etcdctl     | --key                        | --cert                                    |

Aceleași considerente se aplică și pentru perechea de chei a contului de serviciu:

| cale cheie privată            | cale cheie publică           | comanda                 | argument                             |
|------------------------------|-----------------------------|-------------------------|--------------------------------------|
|  sa.key                      |                             | kube-controller-manager | service-account-private              |
|                              | sa.pub                      | kube-apiserver          | service-account-key                  |

## Configurarea certificatelor pentru conturile de utilizator

Trebuie să configurați manual aceste conturi de administrator și conturi de servicii:

| nume de fișier                | nume de acreditare         | Mod implicit CN                     | O (în Subiect) |
|-------------------------|----------------------------|--------------------------------|----------------|
| admin.conf              | default-admin              | kubernetes-admin               | system:masters |
| kubelet.conf            | default-auth               | system:node:`<nodeName>` (see note) | system:nodes   |
| controller-manager.conf | default-controller-manager | system:kube-controller-manager |                |
| scheduler.conf          | default-scheduler          | system:kube-scheduler          |                |

{{< note >}}
Valoarea `<nodeName>` pentru `kubelet.conf` **trebuie**  să se potrivească cu exactitate cu numele nodului furnizat de kubelet așa cum se înregistrează la apiserver. Pentru detalii suplimentare, citiți secțiunea [Node Authorization](/docs/reference/access-authn-authz/node/).
{{< /note >}}

1. Pentru fiecare configurare, generați o pereche x509 cert/cheie  cu CN-ul respectiv și O .

1. Rulați `kubectl` după cum urmează pentru fiecare configurare:

```shell
KUBECONFIG=<filename> kubectl config set-cluster default-cluster --server=https://<host ip>:6443 --certificate-authority <path-to-kubernetes-ca> --embed-certs
KUBECONFIG=<filename> kubectl config set-credentials <credential-name> --client-key <path-to-key>.pem --client-certificate <path-to-cert>.pem --embed-certs
KUBECONFIG=<filename> kubectl config set-context default-system --cluster default-cluster --user <credential-name>
KUBECONFIG=<filename> kubectl config use-context default-system
```

Aceste fișiere sunt utilizate astfel:

| nume de fișier                | comanda                 | cometariu                                                               |
|-------------------------|-------------------------|-----------------------------------------------------------------------|
| admin.conf              | kubectl                 | Configurează utilizatorul de administrator pentru cluster                                     |
| kubelet.conf            | kubelet                 | Unul necesar pentru fiecare nod din cluster.                           |
| controller-manager.conf | kube-controller-manager | Trebuie adăugat în manifesta  `manifests/kube-controller-manager.yaml` |
| scheduler.conf          | kube-scheduler          | Trebuie adăugat în manifesta `manifests/kube-scheduler.yaml`          |


