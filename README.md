# Système de Gestion des Commandes d'Équipements Internes


## Contexte et problématique métier

Ce projet a été conçu et réalisé dans le cadre d'un exercice technique de **5 heures**. L'objectif est de moderniser et d'automatiser le processus de demande d'équipements informatiques et d'accessoires (casques, claviers, souris...) pour une équipe opérationnelle.

### Situation Initiale (Le Problème)
Une équipe des opérations recevait des demandes internes d'équipements (casques, souris, claviers...) et gérait l'état des stocks manuellement au sein d'un **fichier Excel partagé**. Ce processus souffrait de trois failles systémiques :
1. **Échec de Concurrence (*File Locking*) :** Excel appliquant un verrouillage exclusif à l'écriture en mode lourd, dès qu'un gestionnaire ouvrait le fichier pour modifier une ligne, les autres membres de l'équipe étaient relégués en "Lecture seule".
2. **Opacité Opérationnelle :** Les employés demandeurs n'avaient aucun canal pour suivre le cycle de vie de leur commande (Nouvelle ➡️ Approuvée ➡️ Clôturée), ce qui générait une surcharge d'emails de relance pour l'équipe logistique.
3. **Risque d'Intégrité des Données :** L'absence de barrières de saisie relationnelle laissait la porte ouverte aux erreurs de frappe, aux références obsolètes et aux doublons.

### Stratégie de Délivrance (L'arbitrage de la Timebox)
Pour concevoir une solution viable en 5 heures, une distinction claire a été faite entre l'essentiel et le superflu :
* **Priorités :** Modélisation d'une base de données cloud relationnelle solide, sécurité des écritures multi-utilisateurs et fluidité du cycle de vie des requêtes.
* **Compromis assumé :** Choix d'une interface **Canvas Mobile-First** (format Téléphone). Ce choix répond à la réalité terrain (les gestionnaires se déplacent dans les stocks pour vérifier le matériel) et a évité de consommer du temps précieux sur du design *responsive* desktop complexe.
  


## 2. Architecture Technique & Modèle de Données

Le système abandonne Excel comme stockage actif pour migrer vers une architecture cloud centralisée basée sur **Microsoft Dataverse**. L'ancien fichier Excel a été traité comme une source de données d'initialisation (**Seed Data**), nettoyé, puis déconnecté de la production pour éradiquer les silos.

### Spécifications du Schéma Relationnel (Relation 1:N)

Le modèle s'appuie sur une stricte séparation sémantique entre les données de référence fixes (le catalogue) et les flux de mouvements (les commandes de l'historique). Il s'articule exclusivement autour de deux tables connectées de manière logique :

#### 1. Table Parente : `Equipment_Inventory` (Le Catalogue)
Cette table centralise et protège l'état de référence des consommables de l'entreprise. Chaque ligne contient (toutes les colonnes ne seront pas mentionnées car il y en a plus de 20) :
* `Equipment_InventoryID` *(Clé Primaire - GUID)* : Identifiant unique universel généré automatiquement par Dataverse à la création.
* `Model` *(Texte)* : Modèle de l'équipement (ex: *ADAPT 165*).
* 'Notes' *(Texte)* : Description du produit qui permet de le différencier des autres produits qui peuvent sinon sembler similaires.
* `Category` *(Choix / Option Set)* : Segment logistique de tri (Périphériques, Écrans, Audio).
* `AvailableQty` *(Nombre entier)* : État théorique du stock disponible mis à jour en temps réel.

#### 2. Table Enfant : `Equipment_Orders` (Le Flux Transactionnel)
Cette table journalise l'intégralité des intentions et des cycles de vie des requêtes des collaborateurs. Chaque ligne contient :
* `Equipment_OrdersID` *(Clé Primaire - GUID)* : Identifiant unique de la transaction.
* `OrderTitle` *(Texte)* : Indexation simplifiée contenant généralement le nom de la commande.
* `Quantity` *(Nombre entier)* : Volume unitaire sollicité par le demandeur (avec règle stricte supérieure à zéro).
* `UserEmail` *(Texte)* : Adresse email de la session de l'appelant, capturée de manière immuable à la soumission.
* `NeededByDate` *(Date)* : Date limite de livraison requise par l'employé.
* `Status` *(Choix / Option Set)* : État d'avancement de la demande (`New`, `Approved`, `Fulfilled & Closed`, `Cancelled`).

#### Le Mécanisme de Liaison : Le Champ Lookup
La table enfant `Equipment_Orders` intègre une colonne de type **Lookup (Recherche)** pointant directement vers la table parente `Equipment_Inventory`. 

Cette relation de **1 à N (One-to-Many)** signifie techniquement qu'un même produit du catalogue peut être lié à plusieurs lignes de commandes au fil du temps, mais qu'une ligne de commande ne peut cibler qu'un seul produit valide existant. Ce mécanisme d'intégrité référentielle interdit les "commandes fantômes" et supprime nativement tout risque d'erreur de frappe de la part de l'utilisateur.

### Justification du Data Store : Dataverse vs Excel/SharePoint
Face aux exigences de co-édition et de robustesse face aux accès simultanés, Dataverse s'impose techniquement :

* **Gestion de la concurrence :** Excel et SharePoint souffrent de blocages complets (*File locking*) ou de goulots d'étranglement (*Throttling*) sous forte charge. Dataverse assure une conformité **ACID** en isolant chaque transaction au niveau exact de la ligne modifiée.
* **Intégrité des types :** Contrairement aux dossiers SharePoint ou aux cellules Excel qui acceptent de stocker du texte là où on attend un nombre, Dataverse applique un typage fort, protégeant la qualité globale des données récoltées.
* **Performance à l'échelle :** SharePoint affiche des alertes de délégation pénalisantes au-delà de 2 000 lignes. Dataverse gère de manière native et fluide des volumes de plusieurs millions d'enregistrements.



## 3. Atténuation des Risques de Concurrence, Doublons & Données Obsolètes

Le cœur de cet exercice repose sur la démonstration d'une compréhension des risques de collision de données en environnement multi-utilisateur.

### A. Isolation au niveau de la ligne (Row-Level Isolation)
Grâce au moteur de base de données transactionnel de Dataverse, si Emna et Valentin ouvrent l'application en même temps et modifient deux commandes différentes, le serveur cloud traite et isole les requêtes de manière indépendante. Aucune interaction destructrice ou verrouillage global de la table ne peut survenir.

### B. Idempotence des écritures via la fonction `Patch()`
Pour modifier le statut d'une commande, l'application exclut les formulaires standards d'édition (`SubmitForm`), lourds en bande passante (*payload*) et sujets aux conflits d'états. La mise à jour est déléguée à une commande micro-ciblée via la fonction **`Patch()`** exécutée sur l'événement `OnChange` du sélecteur de statut :

Patch(Equipment_Orders; ThisItem; {Status: Self.Selected.Value})

**Cette syntaxe est sécurisée** car le mot-clé ThisItem transmet directement au serveur cloud le GUID (Global Unique Identifier) immuable de l'enregistrement. L'action devient idempotente : même si un gestionnaire clique de manière compulsive sur le bouton en raison d'une latence réseau, Dataverse comprend qu'il doit appliquer une mise à jour (Upsert) sur l'adresse mémoire de cette ligne précise, éliminant tout risque de duplication accidentelle de ligne.

### C. Neutralisation du risque de données obsolètes
Dans un flux multi-utilisateur, un gestionnaire pourrait valider une commande sur la base d'informations affichées obsolètes (si un autre collègue a modifié le statut en arrière-plan sans que son écran ne soit mis à jour). Pour casser ce cache local, une stratégie applicative de Cache-Busting a été implémentée :

Mécanisme : La propriété OnVisible de l'écran du Cockpit Opérationnel exécute la formule :

Refresh(Equipment_Orders)

**Impact :** À chaque fois qu'un utilisateur bascule sur cet écran, l'application invalide instantanément son cache local et force un appel asynchrone vers Dataverse pour afficher la stricte "Source Unique de Vérité".



## 4. Périmètre Fonctionnel & Expérience Utilisateur
L'application implémente une gestion de profils étanches (simulation de rôles) pour fluidifier les parcours utilisateurs :

### A. Espace employé (Demandeur)
Interface de Création : Un catalogue clair sous forme de galerie permet de parcourir le matériel. L'adresse email de la session de l'utilisateur connecté est automatiquement récupérée par le système et verrouillée en affichage pour empêcher toute usurpation d'identité.

Interface de Suivi : Un tableau de bord personnel filtre les requêtes de l'appelant et les classe par statut (Nouvelles, Approuvées, Clôturées) avec un code couleur explicite (Vert pour les succès, Rouge pour les annulations).

### B. Interface de pilotage opérationnel (Gestionnaire)
Visualisation KPI : Un graphique en secteurs (Pie Chart) affiche en temps réel la répartition de la charge de travail par statut, permettant de repérer instantanément les dossiers prioritaires.

**Gestion du cycle de vie :** Les statuts de commande sont modifiables en ligne à l'aide de contrôles de choix sécurisés reprenant les énumérations (Option Sets, mentionné plus tôt dans la colonne Status de la table Equipment_Orders) de Dataverse, éliminant toute saisie libre de texte.



## 5. Vision cible et roadmap V2 
Conformément aux contraintes du projet (livrer un noyau stable en 5 heures), les processus d'arrière-plan lourds ont été volontairement exclus du frontend de l'application pour être intégrés dans une architecture cible industrialisée dans le futur si jugés utiles :

### Plus de foncionnalités sur l'interface de l'application : 
Intégration de plus de liberté pour les utilisateurs sur leurs commandes (modifier, supprimer avant validation par l'équipe opérationelle), envoi de notifications à l'employé pour l'informer du changement de statut de sa commande, mots de passe pour se connecter, robustesse dans tous les champs (certains le sont mais pas tous encore) pour éviter de rentrer n'importe quelle valeur en toute impunité... etc

### Pipeline d'ingestion de catalogues fournisseurs (Python / Pandas) : 
Les fichiers d'inventaire reçus des constructeurs externes étant souvent altérés ou mal typés, un script Python utilisant la bibliothèque Pandas réalisera les opérations d'ETL (nettoyage, validation de schéma) avant d'injecter proprement les données dans l'inventaire via l'API Web de Dataverse.

### Analyses budgétaires décisionnelles (par exemple, mais on peut faire des dashboards sur d'autres thématiques) Power BI :
Exploitation du connecteur natif Dataverse (DirectQuery) pour déployer des rapports décisionnels à destination de la direction financière, assurant le suivi des coûts et l'analyse prédictive des ruptures de stocks.



## 6. Guide d'Importation & d'Exploration dans Power Apps Studio
Le fichier .msapp stocké dans ce dépôt est un package binaire propriétaire. Il ne peut pas être ouvert localement sur votre ordinateur et doit être importé dans votre environnement de développement.

### Prérequis
- Un compte Microsoft Power Apps (Professionnel, Scolaire ou un Plan Développeur gratuit).

- Avoir téléchargé le fichier GE_Healthcare_app.msapp depuis ce dépôt GitHub sur votre machine.

### Étapes pas à pas pour ouvrir l'application
- Connectez-vous sur le portail d'administration de la Power Platform : make.powerapps.com.

- Vérifiez dans le coin supérieur droit que vous êtes positionné dans votre environnement de Sandbox ou de Développement dédié.

- Dans la barre de navigation gauche, cliquez sur le bouton + Créer (+ Create), puis sélectionnez Application canevas vide (Blank canvas app).

- Spécifiez le format Téléphone, nommez l'application de manière temporaire (ex: Review_Studio) et validez en cliquant sur Créer.

- Une fois que l'environnement de conception (Power Apps Studio) a terminé son chargement, dirigez-vous dans le coin supérieur gauche et cliquez sur Fichier.

- Dans le menu vertical de gauche, cliquez sur Ouvrir, puis sélectionnez l'option Parcourir les fichiers.

- Naviguez dans vos fichiers locaux, sélectionnez le fichier GE_Healthcare_app.msapp précédemment téléchargé, et validez.

- L'outil va désarchiver le fichier source. L'arborescence des écrans, les éléments graphiques et l'intégralité des expressions Power Fx (Patch, Refresh, règles de visibilité conditionnelle) sont désormais pleinement ouverts à l'inspection. Vous pouvez prévisualiser l'expérience utilisateur en appuyant sur la touche F5.

**Note d'environnement :** Les tables de la base de données Dataverse étant rattachées de manière sécurisée à mon locataire informatique de test, les sources de données apparaîtront logiquement déconnectées lors de votre première ouverture. Toute l'intelligence algorithmique et la logique structurelle de l'application restent néanmoins visualisables et auditables via la barre de formule de chaque composant.
