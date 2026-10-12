# Spec Delta

## Purpose

Reconnaître les gestes à plusieurs doigts sur le trackpad (tap, balayage, pincement) et les relier à des actions choisies par l'utilisateur, sans interférer avec les gestes que macOS possède déjà.

## ADDED Requirements

### Requirement: Feature opt-in installable
Les gestes du trackpad DOIVENT être une feature du hub Features, dans le groupe Souris et clavier, non installée par défaut. Une fois installée, son interrupteur principal est activé à la première installation et ses réglages apparaissent dans la page Souris.

#### Scenario: Mise à jour sans changement visible
- **WHEN** un utilisateur met l'app à jour sans jamais avoir installé la feature
- **THEN** aucun réglage de gestes n'apparaît et aucune lecture du trackpad n'est démarrée pour cette feature

#### Scenario: Installation depuis le hub
- **WHEN** l'utilisateur installe Gestes du trackpad depuis Features
- **THEN** la feature est activée, sa carte apparaît dans la page Souris et la permission Accessibilité est demandée si elle manque

#### Scenario: Désinstallation
- **WHEN** l'utilisateur désinstalle la feature
- **THEN** la reconnaissance des gestes s'arrête, les mappages restent sauvegardés et réapparaissent à la réinstallation

### Requirement: Démarrage conditionné à la permission et à la session
La reconnaissance DOIT tourner seulement quand la feature est installée et activée, qu'Accessibilité est accordée et que la session utilisateur est active à l'écran.

#### Scenario: Permission accordée après coup
- **WHEN** l'utilisateur accorde Accessibilité alors que la feature est activée
- **THEN** la reconnaissance démarre sans redémarrer l'app

#### Scenario: Changement rapide d'utilisateur
- **WHEN** la session passe en arrière-plan
- **THEN** la lecture du trackpad et l'exécution d'actions s'arrêtent jusqu'au retour de la session

#### Scenario: Mode nettoyage
- **WHEN** le mode nettoyage est actif
- **THEN** aucun geste n'est reconnu ni exécuté

### Requirement: Gestes reconnus
Le système DOIT reconnaître, à la levée de tous les doigts, au plus un geste par contact parmi : tap à 3, 4 ou 5 doigts ; balayage à 3, 4 ou 5 doigts vers la gauche, la droite, le haut ou le bas ; pincement à 4 ou 5 doigts en fermant ou en ouvrant.

#### Scenario: Tap bref et immobile
- **WHEN** quatre doigts touchent le trackpad moins de 0,35 s sans déplacement ni pression physique
- **THEN** le geste « tap 4 doigts » est émis une fois

#### Scenario: Balayage dominant
- **WHEN** trois doigts se déplacent ensemble vers la droite au-delà du seuil de balayage, l'axe horizontal dominant l'axe vertical
- **THEN** le geste « balayage 3 doigts vers la droite » est émis une fois à la levée des doigts

#### Scenario: Pincement
- **WHEN** cinq doigts se rapprochent au-delà du seuil d'écartement tandis que leur centre se déplace moins que l'écartement ne change
- **THEN** le geste « pincement 5 doigts fermer » est émis une fois à la levée des doigts

#### Scenario: Contact ambigu
- **WHEN** un contact se déplace et change d'écartement dans des proportions voisines, sans que l'un l'emporte nettement sur l'autre, ou qu'un doigt supplémentaire se pose en cours de geste
- **THEN** aucun geste n'est émis pour ce contact

### Requirement: Rejet des faux positifs
Le système NE DOIT PAS émettre de geste lorsqu'un bouton du trackpad est pressé pendant le contact, lorsqu'une touche du clavier a été frappée dans les 0,5 s précédentes, lorsque les positions des doigts sont illisibles, ou lorsque le contact dure plus longtemps qu'un geste (repos des doigts).

#### Scenario: Clic physique pendant le contact
- **WHEN** l'utilisateur presse le trackpad avec trois doigts
- **THEN** aucun geste n'est émis et le clic suit son traitement normal (clic du milieu inclus)

#### Scenario: Frappe en cours
- **WHEN** un contact à quatre doigts suit une frappe clavier de moins de 0,5 s
- **THEN** aucun geste n'est émis

#### Scenario: Gestes enchaînés
- **WHEN** deux gestes mappés sur un raccourci clavier se suivent à moins de 0,5 s
- **THEN** les deux actions sont exécutées : le raccourci envoyé par le premier n'est pas compté comme une frappe

#### Scenario: Données de touche inattendues
- **WHEN** une position lue sort de la plage normalisée du trackpad
- **THEN** le contact est ignoré en entier plutôt que de produire un geste erroné

### Requirement: Mappage geste → action
Chaque geste reconnu DOIT pouvoir être associé à au plus une action du catalogue, ou laissé sans action. Un geste sans action n'a aucun effet. Les mappages DOIVENT persister entre les lancements et participer à l'export et à l'import des réglages.

#### Scenario: Geste mappé
- **WHEN** l'utilisateur associe « balayage 4 doigts vers le haut » à Mission Control et effectue ce geste
- **THEN** l'action Mission Control est exécutée une fois

#### Scenario: Geste non mappé
- **WHEN** l'utilisateur effectue un geste sans action associée
- **THEN** rien ne se produit

#### Scenario: Export et import des réglages
- **WHEN** l'utilisateur exporte ses réglages puis les importe sur un autre Mac
- **THEN** les mappages de gestes sont restaurés et les entrées invalides sont ignorées

### Requirement: Exclusions par application
L'utilisateur DOIT pouvoir lister des applications dans lesquelles aucun geste n'agit. Un geste n'agit pas quand l'application au premier plan ou celle sous le pointeur, au moment de la levée des doigts, est dans la liste, comme pour les raccourcis des boutons de souris.

#### Scenario: App exclue au premier plan
- **WHEN** une app de la liste d'exclusion est au premier plan et un geste mappé est effectué
- **THEN** aucune action n'est exécutée

### Requirement: Conflits avec les gestes macOS
Pour chaque geste que macOS reconnaît lui aussi, les réglages DOIVENT indiquer qu'il est pris par le système et où le libérer dans Réglages Système. Le système NE DOIT PAS tenter de bloquer ni de supprimer un geste système.

#### Scenario: Balayage 3 doigts réservé à Spaces
- **WHEN** le balayage horizontal à trois doigts est réglé sur Spaces dans Réglages Système
- **THEN** la ligne « balayage 3 doigts gauche/droite » est marquée comme prise par macOS avec le chemin Réglages Système, Trackpad, Autres gestes

#### Scenario: Glisser à trois doigts actif
- **WHEN** le glisser à trois doigts d'Accessibilité est activé
- **THEN** tous les gestes à trois doigts sont marqués comme pris par macOS et le chemin Accessibilité, Contrôle du pointeur, Options du trackpad est indiqué

#### Scenario: Geste libéré
- **WHEN** l'utilisateur désactive le geste système correspondant et revient dans les réglages
- **THEN** la marque disparaît sans redémarrage

### Requirement: Cohabitation avec le clic du milieu et le menu radial
Le tap de clic du milieu (3 ou 4 doigts) et le tap 4 doigts du menu radial DOIVENT continuer à fonctionner comme avant. Un nombre de doigts déjà utilisé par l'un d'eux DOIT être marqué comme pris dans les réglages des gestes et ne DOIT PAS déclencher de geste mappé.

#### Scenario: Tap 4 doigts réservé au menu radial
- **WHEN** un profil du menu radial s'ouvre par tap 4 doigts
- **THEN** la ligne « tap 4 doigts » des gestes est marquée comme prise par le menu radial et un tap 4 doigts ouvre le menu radial, sans autre action

#### Scenario: Clic du milieu désactivé
- **WHEN** le clic du milieu et son tap sont désactivés
- **THEN** le tap 3 doigts redevient disponible pour les gestes

### Requirement: Trackpad absent ou lecture indisponible
Quand aucun trackpad n'est lisible, les réglages DOIVENT l'indiquer et la feature DOIT rester inactive sans erreur. Le branchement ou le débranchement d'un trackpad DOIT être pris en compte sans redémarrage.

#### Scenario: Aucun trackpad
- **WHEN** la feature est activée sur un Mac sans trackpad lisible
- **THEN** la carte des réglages affiche que les toucher du trackpad ne peuvent pas être lus

#### Scenario: Magic Trackpad connecté
- **WHEN** un Magic Trackpad se connecte pendant que la feature est active
- **THEN** ses gestes sont reconnus sans redémarrage et l'avertissement disparaît

#### Scenario: Framework système modifié
- **WHEN** macOS ne fournit plus l'API de lecture des contacts
- **THEN** la feature reste simplement inactive, sans plantage

### Requirement: Localisation
Tous les textes de la feature DOIVENT exister dans les quinze langues de l'app, y compris les noms de gestes, les états de conflit et les descriptions du hub.

#### Scenario: Langue sans traduction
- **WHEN** une chaîne manque pour une langue
- **THEN** la compilation ou les tests de localisation échouent
