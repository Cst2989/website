---
reviewers:
- andreipetcu
title: Validarea configurararii nodului
weight: 30
---


## Testul de conformare a nodului

*Testul conformității nodului* este un cadru de testare containerizat care furnizează un sistem de
verificare și de funcționalitate pentru un nod. Testul validează dacă
nodul îndeplinește cerințele minime pentru Kubernetes; un nod care trece testul
este calificat să se alăture unui cluster Kubernetes.

## Limitări

În versiunea 1.5 a lui Kubernetes, test de conformitate a nodului are următoarele limitări:

* Testul conformității nodului acceptă numai Docker ca timp de rulare a containerului.

## Condiție prealabilă a nodului

Pentru a rula testul de conformare a nodului, un nod trebuie să satisfacă aceleași condiții preliminare ca și
nodul standard Kubernetes. Cel puțin, nodul trebuie să aibă următoarele
daemon-uri instalate:

* Container Runtime (Docker)
* Kubelet

## Rularea testului de conformare a nodurilor

Pentru a rula testul conformității nodului, efectuați următorii pași:

1. Indicați-vă Kubelet-ul către localhost `--api-servers="http://localhost:8080"`,
deoarece cadrul de testare pornește un master local care să testeze Kubelet. Sunt cateva
alte flag-uri Kubelet de care vă poate interesa:
  * `--pod-cidr`: Dacă folosiți `kubenet`, ar trebui să specificați un CIDR arbitrar
    la Kubelet, de exemplu `--pod-cidr=10.180.0.0/24`.
  * `--cloud-provider`: Dacă folosiți `--cloud-provider=gce`, ar trebui să
    scoateți steagul pentru a rula testul.

2. Executați testul conformității nodului cu comanda:

```shell
# $CONFIG_DIR este calea manifestă a podului din Kubelet.
# $LOG_DIR este calea de output a testului.
sudo docker run -it --rm --privileged --net=host \
  -v /:/rootfs -v $CONFIG_DIR:$CONFIG_DIR -v $LOG_DIR:/var/result \
  k8s.gcr.io/node-test:0.2
```

## Rularea testului de conformitate a nodurilor pentru alte arhitecturi

Kubernetes oferă, de asemenea, imagini de docker pentru testarea conformității nodului
pe alte arhitecturi:

  Arch  |       Image       |
--------|:-----------------:|
 amd64  |  node-test-amd64  |
  arm   |    node-test-arm  |
 arm64  |  node-test-arm64  |

## Rularea testul selectat

Pentru a rula teste specifice, suprascrieți variabila de mediu `FOCUS` cu
expresia regulată a testelor pe care doriți să le executați.

```shell
sudo docker run -it --rm --privileged --net=host \
  -v /:/rootfs:ro -v $CONFIG_DIR:$CONFIG_DIR -v $LOG_DIR:/var/result \
  -e FOCUS=MirrorPod \ # Ruleaza numai testul MirrorPod
  k8s.gcr.io/node-test:0.2
```

Pentru a omite testele specifice, rescrieți variabila de mediu `SKIP` cu
expresia regulară a testelor pe care doriți să săriți.

```shell
sudo docker run -it --rm --privileged --net=host \
  -v /:/rootfs:ro -v $CONFIG_DIR:$CONFIG_DIR -v $LOG_DIR:/var/result \
  -e SKIP=MirrorPod \ # Executați toate testele de conformitate, dar săriți testul MirrorPod
  k8s.gcr.io/node-test:0.2
```
Testul conformității nodului este o versiune containerizată a [node e2e test](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/e2e-node-tests.md).
În mod implicit, acesta rulează toate testele de conformitate.

Teoretic, puteți rula orice test e2e dacă configurați containerul și
montați volumele necesare în mod corespunzător. Dar **este recomandat să se execute numai teste de
conformitate**, deoarece necesită o configurație mult mai complexă pentru a rula teste non-conformitationale.

## Caveats

* Testul lasă câteva imagini pe nod, inclusiv imaginea testului de conformitatea a nodului
  și imaginile containerelor utilizate în funcționalitatea testului.
* Testul lasă containerele moarte pe nod. Aceste containere sunt create
  în timpul testului de funcționalitate.
