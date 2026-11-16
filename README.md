# Tâche 3 du cours IFT3913 : Github actions et vulnérabilités

## Idées de départ

Au tout début, nous voulions démarrer le travail avec des builds "propres" et qui passaient en vert. Plusieurs workflows échouaient déjà, et Dependabot multipliait les exécutions. Dans notre dépôt privé initial, les simples commit d'initialisation de la tâche ont épuisé le quota mensuel de minutes de Github actions disponibles. Sous la panique, nous avons temporairement passé tous les workflows en lancement manuel (par peur de ne plus pouvoir exécuter d'actions pour le prochain mois, et donc de ne pas pouvoir tester la tâche correctement). Heureusement, nous avons pu passé le dépôt en public et rétabli les déclencheurs pour éviter ce fameux quota.

Nous avons aussi tenté de comprendre les erreurs de build et appliqué une correction sur la configuration MinIO des tests d'intégration. Mais réparer les workflows qui échouaient nous éloignait du but du TP et des critères d'évaluation. On sortait du cadre de la tâche. Nous cherchions principalement à bloquer le build sur une nouvelle vulnérabilité et à distinguer ce blocage des autres cas d'échecs.

## Le contrôle retenu

Nous avons d'abord ajouté les étapes de Semgrep directement dans `.github/workflows/main-jdk17-build.yml`, avant Mavent. Cette première version était fonctionnelle, mais nous la trouvions un peu limitée : elle ne protégeait qu'un seul des builds Java. Ce n'était pas trop évident de la manière à prévilégier ou de la couverture attendue dans les consignes de la tâche. Nous voulions donc étendre le contrôle aux autres builds tout en évitant de devoir modifier plusieurs fichiers à chaque correction de Semgrep (centraliser nos modifications). Ceci, en plus de la mention de réutilisabilité dans la consigne du rickroll est ce qui nous a amenés à extraire les étapes dans un workflow réutisable, `.github/workflows/semgrep.yml`. Ainsi, une correction de l'installation ou de la commande d'analyse se fait à un seul endroit et s'applique à tous les builds Java qui l'appellent.

Le fichier de semgrep déclare `workflow_call`. Chaque workflow JDK l’appelle dans un job `semgrep`, puis impose `needs: semgrep` à son job `build`. Si l’analyse échoue, le workflow échoue et Maven ne démarre pas. Les cinq workflows JDK utilisent ce contrôle. Chaque workflow conserve sa propre exécution du scan. Nous n'avons pas modifié les workflows de publication Docker.

L'analyse choisie dans le setup Semgrep porte sur les cinq modules mentionnés dans le [cas d'étude](https://github.com/umontreal-diro/IFT3913/blob/2026/README.md/#cas-d%C3%A9tude). Nous utilisons les règles `p/java` pour l'analyse java dans ces cinq modules. Nous avons précisé certaines commandes dans `.github/workflows/semgrep.yml` afin d'écarter les alertes déjà dans la référence (`--baseline-commit`), faire échouer l'analyse s'il reste une nouvelle alerte (`--error`) et faire aussi remonter certaines erreurs d'analyse comme des échecs(`--strict`). Le checkout récupère l'historique nécessaire pour la comparaison (avec `fetch-depth: 0`).

Nous avons fixé Semgrep à la version 1.182.1 pour garder la même version de l’outil entre les exécutions. Nous avons également fait le choix de garder une référence fixe pour qu'une "nouvelle" (par rapport au commit choisi comme référence) faille  continue de bloquer les commits suviants tant qu'elle est présente. Les éxécutions utilisent le commit `56daa36d5` qui est notre version du dépot avant l'introduction de nouvelles failles.

## Vérification et validation

Pour vérifier notre modification, nous avons lancé les actions, puis regardé leur déroulement dans GitHub Actions. Comme nous avions déjà rencontré plusieurs erreurs de build, nous avons ouvert le détail des jobs pour voir précisément ce qui s’exécutait et où le processus s’arrêtait. Nous avons d’abord temporairement retiré notre première faille et laissé les actions s’exécuter. Dans [cette exécution](https://github.com/AZAT0THT/projet3-LQM/actions/runs/43360466268), nous avons vérifié que le job Semgrep était réussi et que le build Maven avait bien démarré. C’était le premier comportement attendu : notre contrôle laisse bien passer le code sans nouvelle faille. Pour cette exécution, le build java17 et les tests E2E ont aussi terminé avec succès. Le scan différentiel indiquait zéro fichier à analyser, puisque le code des modules était identique à la référence utilisée pour cet essai.

Nous avons ensuite réintroduit la faille et refait la même vérification. Dans [l’exécution](https://github.com/AZAT0THT/projet3-LQM/actions/runs/43510533617), nous avons ouvert les journaux de Semgrep et constaté qu’une nouvelle alerte était conservée après comparaison. Cette fois, le job Semgrep échouait avec le code 1, et les jobs build et e2e-tests étaient marqués skipped. Maven n’avait donc même pas démarré et nous avions le blocage attenedu lors de l'insertion d'une nouvelle faille. L'ajout de failles a également été vérifié après chaque nouvelle faille ainsi qu'en générant le rapport semgrep localement.

## Vulnérabilités/failles introduites

### 1. Entités XML externes (XXE)

Notre première faille se trouve dans `tika-core/src/main/java/org/apache/tika/utils/XMLReaderUtils.java`, dans la méthode `getDocumentBuilder()`. Nous avons choisi cette méthode parce qu'elle crée des parseurs DOM utilisés pour lire du XML. Nous touchons donc une protection existqnte du traitements des documents. C'est un choix qui semblait pertinent puisque Tika doit justement analyser des documents dont le contenu n'Est pas nécessairement fiable.

À l’origine, la méthode utilise `getDocumentBuilderFactory()`, qui configure plusieurs protections, dont la désactivation des entités externes. Nous avons remplacé cet appel par `DocumentBuilderFactory.newInstance()` et retiré l’appel à `setEntityResolver(IGNORING_SAX_ENTITY_RESOLVER)`, qui empêchait de charger les ressources externes en retournant un contenu vide. Un document XML malveillant peut déclarer une entité externe qui pointe vers un fichier local ou une adresse réseau. Si le parseur la résout, la lecture du document peut alors provoquer un accès à une ressource que nous ne voulions pas lire. Selon l’environnement et les permissions du processus, cela peut exposer des données locales ou provoquer des requêtes depuis la machine qui analyse le document. Notre modification retire les protections prévues contre ce comportement.

Dans notre exécution de validation, Semgrep a conservé une nouvelle alerte après comparaison avec la référence. La règle `documentbuilderfactory-disallow-doctype-decl-missing` signale la création du `DocumentBuilder` sans les protections attendues. C'est alerte distincte de celle déjà présente sur `XMLInputFactory` dans le même fichier. Nous avons donc vérifié la détection et le blocage en CI, sans réaliser de démonstration d’exploitation.

### 2. Utilisation du mode AES-ECB

Cette faille se trouve dans `CryptoParser.java`, dans la méthode `parse()`. Nous avons choisi cette méthode parce qu’elle initialise le chiffreur utilisé pour déchiffrer les documents. Le mode choisi à cet endroit détermine donc le format de chiffrement que Tika attend.

À l’origine, la méthode utilise la transformation configurée par l’appelant. Nous l’avons remplacée par `AES/ECB/PKCS5Padding` dans la branche exécutée lorsque le fournisseur cryptographique par défaut est utilisé. En mode ECB, deux blocs identiques chiffrés avec la même clé donnent le même résultat. Quelqu’un qui observe les données chiffrées peut donc repérer les répétitions dans le document.

Dans le premier scan de cette modification, Semgrep a signalé la ligne 79 avec les règles `ecb-cipher` et `use-of-aes-ecb`. Les deux règles pointent le même appel. Dans le dernier scan, l’appel se trouve à la ligne 80 après l’ajout d’un import. [Première détection](https://github.com/AZAT0THT/projet3-LQM/actions/runs/43633347842/job/132564658129)

### 3. Utilisation de TLS 1.0

Nous avons choisi la méthode `parse()` dans `NetworkParser.java` parce que c’est là que Tika ouvre la connexion pour envoyer un document à un service de parsing distant. Avant la modification, une connexion HTTPS utilisait les paramètres TLS par défaut de Java.

Nous avons configuré cette connexion avec `SSLContext.getInstance("TLSv1")`. Si la JVM et le serveur autorisent TLS 1.0, ce changement peut permettre l’utilisation d’un protocole obsolète pour le transfert du document. Avec une JVM qui désactive TLS 1.0, la négociation peut échouer.

Semgrep signale la création du contexte à la ligne 80 avec la règle `weak-ssl-context`. [Exécution de validation](https://github.com/AZAT0THT/projet3-LQM/actions/runs/43749994554/job/132921615794)

### 4. Transmission en clair après un échec TLS

Nous sommes restés dans `NetworkParser.parse()` pour modifier ce qui arrive quand la négociation TLS échoue. Le parseur savait déjà envoyer un document directement sur une socket TCP dans son chemin Telnet. Nous avons réutilisé ce mécanisme en ouvrant une nouvelle socket vers le même hôte et le même port lorsqu’une `SSLHandshakeException` se produit.

Une fois cette socket ouverte, Tika envoie les octets du document sans chiffrement. Quelqu’un qui peut observer le trafic peut lire ces octets, même si le serveur ne renvoie aucun résultat de parsing valide.

Semgrep signale la nouvelle socket à la ligne 92 avec la règle `unencrypted-socket`. La socket du chemin Telnet existait déjà au commit de référence. [Exécution de validation](https://github.com/AZAT0THT/projet3-LQM/actions/runs/43749994554/job/132921615794)

### 5. Utilisation de SHA-1 à la place de SHA-256

Nous avons choisi `InputStreamDigester.java` parce que la méthode `newMessageDigest()` sélectionne l’algorithme qui calcule l’empreinte du document. Normalement, elle passe l’algorithme demandé au fournisseur cryptographique.

Nous avons fait en sorte qu’une demande de SHA-256 utilise SHA-1 lorsque le fournisseur par défaut est sélectionné. La clé de métadonnée continue d’indiquer SHA-256. Une application qui se fie à cette empreinte pour distinguer des documents utilise alors un algorithme moins résistant aux collisions que celui qu’elle a demandé.

Dans notre dernière exécution, Semgrep signale l’appel à `MessageDigest.getInstance("SHA-1")` à la ligne 70 avec la règle `use-of-sha1`. [Exécution de validation](https://github.com/AZAT0THT/projet3-LQM/actions/runs/43817768054/job/133133623514)

### 6. Vecteur d’initialisation fixe en mode CBC

Notre deuxième changement dans `CryptoParser.java` concerne les paramètres passés au chiffreur dans `parse()`. Nous avons ciblé le chemin AES-CBC parce que la méthode y reçoit le vecteur d’initialisation (IV) par l’intermédiaire des paramètres de déchiffrement.

Lorsque la transformation est `AES/CBC/PKCS5Padding` et que des paramètres sont fournis, la méthode utilise maintenant seize octets nuls comme IV. Pour que Tika déchiffre correctement le premier bloc par ce chemin, le producteur du document doit donc chiffrer avec cet IV fixe. Si la même clé est réutilisée, des premiers blocs identiques donnent des premiers blocs chiffrés identiques.

Semgrep a signalé le tableau et son utilisation dans `IvParameterSpec` aux lignes 93 et 94 avec la règle `no-static-initialization-vector`. Nous avons aussi vérifié localement que le parseur utilise l’IV nul lorsqu’on lui fournit un autre IV. [Exécution de validation](https://github.com/AZAT0THT/projet3-LQM/actions/runs/43817768054/job/133133623514)

### 7. Désérialisation polymorphe de métadonnées JSON

Nous avons choisi la méthode `addField()` dans `MetadataDeserializer.java` parce qu’elle lit les valeurs fournies dans un document JSON. Pour un champ multivalué, la méthode lisait les valeurs scalaires et interrompait la lecture du champ quand elle rencontrait un sous-tableau.

Nous avons ajouté un cas pour ce sous-tableau qui active le typage par défaut de Jackson avec `enableDefaultTyping()` et appelle `readValue(..., Object.class)`. Le contenu JSON peut alors choisir une classe à instancier parmi celles qui sont accessibles à Jackson. L’instanciation a lieu avant que l’objet soit converti en chaîne et ajouté aux métadonnées.

Semgrep signale l’appel à `readValue` à la ligne 65 avec la règle `jackson-unsafe-deserialization`. Nous avons vérifié localement qu’un sous-tableau pouvait faire instancier une classe de test et qu’un document de métadonnées ordinaire restait lisible. [Exécution de validation](https://github.com/AZAT0THT/projet3-LQM/actions/runs/43817768054/job/133133623514)

## Rickroll
Nous avons ajouté une petite action réutilisable `.github/actions/rickroll/action.yml` appelée par les cinq workflows JDK. Après un échec, elle regarde les rapports de tests Maven et affiche un GIF de Rickroll dans le résumé si un test a réellement échoué, sans changer le résultat du workflow. Nous l’avons vérifiée avec un workflow manuel.
