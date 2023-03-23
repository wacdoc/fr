Entrez dans l'entrepôt de configuration ops.soft, exécutez `./ssl.sh` , et un dossier `conf` sera créé dans **le répertoire supérieur** .

## préambule

SMTP peut acheter directement des services auprès de fournisseurs de cloud, tels que :

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Poussée d'e-mails dans le cloud Ali](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Vous pouvez également créer votre propre serveur de messagerie - envoi illimité, faible coût global.

Ci-dessous, nous montrons étape par étape comment créer notre propre serveur de messagerie.

## Sélection du serveur

Le serveur SMTP auto-hébergé nécessite une adresse IP publique avec les ports 25, 456 et 587 ouverts.

Les clouds publics couramment utilisés ont bloqué ces ports par défaut, et il est peut-être possible de les ouvrir en émettant un ordre de travail, mais c'est très gênant après tout.

Je recommande d'acheter auprès d'un hôte qui a ces ports ouverts et prend en charge la configuration de noms de domaine inversés.

Ici, je recommande [Contabo](https://contabo.com) .

Contabo est un fournisseur d'hébergement basé à Munich, en Allemagne, fondé en 2003 avec des prix très compétitifs.

Si vous choisissez l'euro comme devise d'achat, le prix sera moins cher (un serveur avec 8 Go de mémoire et 4 processeurs coûte environ 529 yuans par an, et les frais d'installation initiaux sont gratuits pendant un an).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Lorsque vous passez une commande, remarquez `prefer AMD` , et le serveur avec processeur AMD aura de meilleures performances.

Dans ce qui suit, je prendrai le VPS de Contabo comme exemple pour montrer comment créer votre propre serveur de messagerie.

## Configuration du système Ubuntu

Le système d'exploitation ici est Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Si le serveur sur ssh affiche `Welcome to TinyCore 13!` (comme indiqué dans la figure ci-dessous), cela signifie que le système n'a pas encore été installé. Veuillez vous déconnecter de ssh et attendre quelques minutes pour vous reconnecter.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Lorsque `Welcome to Ubuntu 22.04.1 LTS` apparaît, l'initialisation est terminée et vous pouvez continuer avec les étapes suivantes.

### [Facultatif] Initialiser l'environnement de développement

Cette étape est facultative.

Pour plus de commodité, j'ai mis l'installation et la configuration système du logiciel ubuntu dans [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Exécutez la commande suivante pour installer en un clic.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Utilisateurs chinois, veuillez utiliser la commande suivante à la place, et la langue, le fuseau horaire, etc. seront automatiquement définis.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo active IPV6

Activez IPV6 pour que SMTP puisse également envoyer des e-mails avec des adresses IPV6.

modifier `/etc/sysctl.conf`

Modifier ou ajouter les lignes suivantes

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Suivez [le didacticiel contabo : Ajouter une connectivité IPv6 à votre serveur](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Modifiez `/etc/netplan/01-netcfg.yaml` , ajoutez quelques lignes comme indiqué dans la figure ci-dessous (le fichier de configuration par défaut de Contabo VPS contient déjà ces lignes, décommentez-les simplement).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Ensuite, `netplan apply` pour que la configuration modifiée prenne effet.

Une fois la configuration réussie, vous pouvez utiliser `curl 6.ipw.cn` pour afficher l'adresse IPv6 de votre réseau externe.

## Cloner les opérations du référentiel de configuration

```
git clone https://github.com/wactax/ops.soft.git
```

## Générez un certificat SSL gratuit pour votre nom de domaine

L'envoi de courrier nécessite un certificat SSL pour le cryptage et la signature.

Nous utilisons [acme.sh](https://github.com/acmesh-official/acme.sh) pour générer des certificats.

acme.sh est un outil de signature de certificat automatisé open source,

Entrez dans l'entrepôt de configuration ops.soft, exécutez `./ssl.sh` , et un dossier `conf` sera créé dans **le répertoire supérieur** .

Trouvez votre fournisseur DNS à partir de [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , modifiez `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Exécutez ensuite `./ssl.sh 123.com` pour générer des certificats `123.com` et `*.123.com` pour votre nom de domaine.

La première exécution installera automatiquement [acme.sh](https://github.com/acmesh-official/acme.sh) et ajoutera une tâche planifiée pour le renouvellement automatique. Vous pouvez voir `crontab -l` , il y a une telle ligne comme suit.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Le chemin du certificat généré est quelque chose comme `/mnt/www/.acme.sh/123.com_ecc。`

Le renouvellement du certificat appellera le script `conf/reload/123.com.sh` , modifiez ce script, vous pouvez ajouter des commandes telles que `nginx -s reload` pour actualiser le cache de certificat des applications associées.

## Créer un serveur SMTP avec chasquid

[chasquid](https://github.com/albertito/chasquid) est un serveur SMTP open source écrit en langage Go.

En tant que substitut des anciens programmes de serveur de messagerie tels que Postfix et Sendmail, chasquid est plus simple et plus facile à utiliser, et il est également plus facile pour le développement secondaire.

Exécutez `./chasquid/init.sh 123.com` sera installé automatiquement en un clic (remplacez 123.com par votre nom de domaine d'envoi).

## Configurer la signature électronique DKIM

DKIM est utilisé pour envoyer des signatures d'e-mails afin d'éviter que les lettres ne soient traitées comme du spam.

Une fois la commande exécutée avec succès, vous serez invité à définir l'enregistrement DKIM (comme indiqué ci-dessous).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Ajoutez simplement un enregistrement TXT à votre DNS (comme indiqué ci-dessous).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Afficher l'état du service et les journaux

 `systemctl status chasquid` Afficher l'état du service.

L'état de fonctionnement normal est comme indiqué dans la figure ci-dessous

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` ou `journalctl -xeu chasquid` peut afficher le journal des erreurs.

## Inverser la configuration du nom de domaine

Le nom de domaine inverse permet de résoudre l'adresse IP en nom de domaine correspondant.

La définition d'un nom de domaine inversé peut empêcher les e-mails d'être identifiés comme spam.

Lorsque le courrier est reçu, le serveur de réception effectuera une analyse de nom de domaine inverse sur l'adresse IP du serveur d'envoi pour confirmer si le serveur d'envoi a un nom de domaine inverse valide.

Si le serveur d'envoi n'a pas de nom de domaine inversé ou si le nom de domaine inversé ne correspond pas à l'adresse IP du serveur d'envoi, le serveur de réception peut reconnaître l'e-mail comme spam ou le rejeter.

Visitez [https://my.contabo.com/rdns](https://my.contabo.com/rdns) et configurez comme indiqué ci-dessous

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Après avoir défini le nom de domaine inversé, n'oubliez pas de configurer la résolution directe du nom de domaine ipv4 et ipv6 sur le serveur.

## Modifier le nom d'hôte de chasquid.conf

Modifiez `conf/chasquid/chasquid.conf` à la valeur du nom de domaine inverse.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Ensuite, exécutez `systemctl restart chasquid` pour redémarrer le service.

## Sauvegarder la configuration dans le référentiel git

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Par exemple, je sauvegarde le dossier conf dans mon propre processus github comme suit

Créez d'abord un entrepôt privé

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Entrez dans le répertoire conf et soumettez-le à l'entrepôt

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Ajouter un expéditeur

courir

```
chasquid-util user-add i@wac.tax
```

Peut ajouter un expéditeur

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Vérifiez que le mot de passe est correctement défini

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Après avoir ajouté l'utilisateur, `chasquid/domains/wac.tax/users` sera mis à jour, n'oubliez pas de le soumettre à l'entrepôt.

## DNS ajouter un enregistrement SPF

SPF (Sender Policy Framework) est une technologie de vérification des e-mails utilisée pour prévenir la fraude par e-mail.

Il vérifie l'identité d'un expéditeur de courrier en vérifiant que l'adresse IP de l'expéditeur correspond aux enregistrements DNS du nom de domaine qu'il prétend être, empêchant les fraudeurs d'envoyer de faux e-mails.

L'ajout d'enregistrements SPF peut empêcher autant que possible que les e-mails soient identifiés comme spam.

Si votre serveur de noms de domaine ne prend pas en charge le type SPF, ajoutez simplement un enregistrement de type TXT.

Par exemple, le SPF de `wac.tax` est le suivant

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF pour `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Notez que j'ai `include:_spf.google.com` ici, car je configurerai ultérieurement `i@wac.tax` comme adresse d'envoi dans la boîte aux lettres Google.

## Configuration DNS DMARC

DMARC est l'abréviation de (Domain-based Message Authentication, Reporting & Conformance).

Il est utilisé pour capturer les rebonds SPF (peut-être causés par des erreurs de configuration, ou quelqu'un d'autre se fait passer pour vous pour envoyer du spam).

Ajouter l'enregistrement TXT `_dmarc` ,

Le contenu est le suivant

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

La signification de chaque paramètre est la suivante

### p (Politique)

Indique comment gérer les e-mails qui échouent à la vérification SPF (Sender Policy Framework) ou DKIM (DomainKeys Identified Mail). Le paramètre p peut être défini sur l'une des trois valeurs suivantes :

* none : aucune action n'est entreprise, seul le résultat de la vérification est renvoyé à l'expéditeur via le mécanisme de rapport par e-mail.
* Quarantaine : placez le courrier qui n'a pas passé la vérification dans le dossier spam, mais ne rejettera pas le courrier directement.
* rejeter : rejeter directement les e-mails qui échouent à la vérification.

### fo (options d'échec)

Spécifie la quantité d'informations renvoyées par le mécanisme de rapport. Il peut être défini sur l'une des valeurs suivantes :

* 0 : Rapporter les résultats de validation pour tous les messages
* 1 : ne signaler que les messages dont la vérification échoue
* d : ne signaler que les échecs de vérification du nom de domaine
* s : signale uniquement les échecs de vérification SPF
* l : signaler uniquement les échecs de vérification DKIM

### rua & ruf

* rua (URI de rapport pour les rapports agrégés) : adresse e-mail pour recevoir les rapports agrégés
* ruf (Reporting URI for Forensic reports) : adresse e-mail pour recevoir des rapports détaillés

## Ajouter des enregistrements MX pour transférer des e-mails vers Google Mail

Parce que je n'ai pas trouvé de boîte aux lettres d'entreprise gratuite prenant en charge les adresses universelles (Catch-All, peut recevoir tous les e-mails envoyés à ce nom de domaine, sans restrictions sur les préfixes), j'ai utilisé chasquid pour transférer tous les e-mails vers ma boîte aux lettres Gmail.

**Si vous avez votre propre boîte aux lettres professionnelle payante, veuillez ne pas modifier le MX et sauter cette étape.**

Modifier `conf/chasquid/domains/wac.tax/aliases` , définir la boîte aux lettres de transfert

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` indique tous les e-mails, `i` est le préfixe de l'adresse e-mail de l'utilisateur expéditeur créé ci-dessus. Pour transférer du courrier, chaque utilisateur doit ajouter une ligne.

Ajoutez ensuite l'enregistrement MX (je pointe directement sur l'adresse du nom de domaine inverse ici, comme indiqué sur la première ligne de la figure ci-dessous).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Une fois la configuration terminée, vous pouvez utiliser d'autres adresses e-mail pour envoyer des e-mails à `i@wac.tax` et `any123@wac.tax` pour voir si vous pouvez recevoir des e-mails dans Gmail.

Si ce n'est pas le cas, vérifiez le journal chasquid ( `grep chasquid /var/log/syslog` ).

## Envoyez un e-mail à i@wac.tax avec Google Mail

Après que Google Mail ait reçu le courrier, j'espérais naturellement répondre avec `i@wac.tax` au lieu de i.wac.tax@gmail.com.

Visitez [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) et cliquez sur "Ajouter une autre adresse e-mail".

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Ensuite, entrez le code de vérification reçu par l'e-mail qui a été transféré.

Enfin, elle peut être définie comme adresse d'expéditeur par défaut (avec la possibilité de répondre avec la même adresse).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

De cette façon, nous avons terminé la mise en place du serveur de messagerie SMTP et utilisons en même temps Google Mail pour envoyer et recevoir des e-mails.

## Envoyez un e-mail de test pour vérifier si la configuration a réussi

Entrez `ops/chasquid`

Exécutez `direnv allow` pour installer les dépendances (direnv a été installé dans le processus d'initialisation à une clé précédent et un crochet a été ajouté au shell)

puis cours

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

La signification des paramètres est la suivante

* utilisateur : nom d'utilisateur SMTP
* pass : mot de passe SMTP
* à : destinataire

Vous pouvez envoyer un e-mail de test.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Il est recommandé d'utiliser Gmail pour recevoir des e-mails de test afin de vérifier si les configurations sont réussies.

### Cryptage standard TLS

Comme le montre la figure ci-dessous, il y a ce petit verrou, ce qui signifie que le certificat SSL a été activé avec succès.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Cliquez ensuite sur "Afficher l'e-mail d'origine"

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Comme le montre la figure ci-dessous, la page de messagerie d'origine de Gmail affiche DKIM, ce qui signifie que la configuration DKIM est réussie.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Cochez la case Reçu dans l'en-tête de l'e-mail d'origine et vous pouvez voir que l'adresse de l'expéditeur est IPV6, ce qui signifie qu'IPV6 est également configuré avec succès.
