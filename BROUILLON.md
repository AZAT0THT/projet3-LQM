
## fichier : CryptoParser.java 
Utilisation du mode AES-ECB — Dans CryptoParser.java, la méthode parse() initialise le chiffreur utilisé pour déchiffrer les documents. 
Nous avons forcé AES/ECB/PKCS5Padding lorsque le fournisseur cryptographique par défaut est utilisé. 
Avec ECB, des blocs identiques chiffrés avec la même clé produisent des blocs chiffrés identiques, ce qui révèle des motifs dans les données.
Semgrep a signalé la ligne 79 avec les règles ecb-cipher et use-of-aes-ecb dans le commit e5a4d3d.
https://github.com/AZAT0THT/projet3-LQM/actions/runs/43633347842/job/132564658129
