# Documentație Kubernetes

[![Status Netlify](https://api.netlify.com/api/v1/badges/be93b718-a6df-402a-b4a4-855ba186c97d/deploy-status)](https://app.netlify.com/sites/kubernetes-io-master-staging/deploys) [![GitHub release](https://img.shields.io/github/release/kubernetes/website.svg)](https://github.com/kubernetes/website/releases/latest)

Acest repository conține resursele necesare să faci build la [Site-ul Kubernetes și documentația](https://kubernetes.io/). Ne bucurăm că vrei să contribui!

## Rularea sitului web local folosind Hugo

Vezi [documentația oficială Hugo](https://gohugo.io/getting-started/installing/) pentru instrucțiuni de instalare Hugo. Asigură-te că instalezi versiunea extinsă Hugo specificată de variabila de mediu `HUGO_VERSION` in fișierul  [`netlify.toml`](netlify.toml#L10)

Pentru a rula situl web local cănd ai Hugo instalat:
```bash
git clone https://github.com/kubernetes/website.git
cd website
git submodule update --init --recursive
hugo server --buildFuture
```
Comenzile acestea vor porni serverul local Hugo pe portul 1313. Deschide browserul la adresa http://localhost:1313 pentru a vizualiza situl. Pe măsură ce faci modificări la fișierele sursă, Hugo actualizează situl și forțează actualizarea browserului.

## Comunitate, discuție, contribuție și sprijin

Află cum să te implici în comunitatea Kubernetes vizitând [pagina comunității](https://github.com/kubernetes/community/tree/master/sig-docs#meetings).

Poți contacta coordonatorii acestui proiect pe:
- [Slack](https://kubernetes.slack.com/messages/sig-docs)
- [Lista de email](https://groups.google.com/forum/#!forum/kubernetes-sig-docs)

## Contribuind la documentație

Poți să dai click pe butonul de  **Fork** din colțul sus-dreapta al ecranului pentru a creea o copie a acestui repository în contul tău de Github. Această copie se numește un *fork*. Poți să faci câte modificari vrei în fork-ul tău, și când ești pregătit să trimiți modificăriile către noi, dute la tine în fork și crează un nou pull request ca să fim notificați.

Odată ce pull requestu-ul tău este creat, un reviewer de la Kubernetes iși asumă responsabilitatea să ofere feedback clar și acționabil. Ca proprietarul pull request-ului, **este responsabilitatea ta să modifici pull request-ul tău ca să adresezi feedback-ul primit de la reviewer-ul Kubernetes.**

Deasemenea, să ții cont că este posibil să primești feedback de la mai mulți reviewers sau să primești feedback de la un reviewer diferit față de cel care ți-a fost asignat.

În plus, în unele cazuri, unul din reviewers poate să ceară un review tehnic de la un Kubernetes tech reviewer când este nevoie. Reviewer-ii o să facă tot posibilul să îți ofere feedback-ul în timp util dar timpul de răspuns poate să varieze de la caz la caz.

Pentru mai multe informații despre contibuția la documentatia Kubernetes:

* [Contribuind la Documentație Kubernetics](https://kubernetes.io/docs/contribute/)
* [Tipuri de Continut de Pagini](https://kubernetes.io/docs/contribute/style/page-content-types/)
* [Ghid de stil de Documentare](https://kubernetes.io/docs/contribute/style/style-guide/)
* [Localizarea Documentației Kubernetes ](https://kubernetes.io/docs/contribute/localization/)

## Localizarea `README.md`-urilor

| Language  | Language |
|---|---|
|[Chineză](README-zh.md)|[Koreană](README-ko.md)|
|[Franceză](README-fr.md)|[Poloneză](README-pl.md)|
|[Germană](README-de.md)|[Portugheză](README-pt.md)|
|[Hindusă](README-hi.md)|[Rusa](README-ru.md)|
|[Indoneză](README-id.md)|[Spaniolă](README-es.md)|
|[Italiană](README-it.md)|[Ukrainiană](README-uk.md)|
|[Japoneză](README-ja.md)|[Vietnameză](README-vi.md)|

## Codul de Conduită

Participarea în comunitatea Kubernetes este guvernată de [CNCF Code of Conduct](https://github.com/cncf/foundation/blob/master/code-of-conduct.md).

## Mulțumim!

Kubernetes prosperă la participarea comunității și apreciem contribuțiile dvs. la site-ul nostru și la documentația noastră!
