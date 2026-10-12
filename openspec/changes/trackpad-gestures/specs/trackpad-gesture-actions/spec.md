# Spec Delta

## Purpose

Définir les actions qu'un geste du trackpad peut déclencher et leur effet observable, en réutilisant le catalogue d'actions du menu radial et en y ajoutant la navigation entre Spaces et vues d'ensemble.

## ADDED Requirements

### Requirement: Catalogue d'actions
Un geste DOIT pouvoir déclencher une action parmi : raccourci clavier, Space précédent, Space suivant, Mission Control, App Exposé, disposition de fenêtre, outil Vorssaint, bascule rapide, touche média, ouvrir une application, ouvrir une URL. Les actions proposées DOIVENT être les mêmes que celles d'une tranche du menu radial, plus les quatre actions de navigation.

#### Scenario: Choix d'une action dans les réglages
- **WHEN** l'utilisateur ouvre le sélecteur d'action d'un geste
- **THEN** il voit ces familles d'actions et peut en choisir une ou laisser le geste sans action

### Requirement: Raccourci clavier
L'action raccourci DOIT envoyer la combinaison enregistrée à l'application au premier plan, après que les modificateurs physiquement maintenus ont été relâchés, comme le font les boutons de souris.

#### Scenario: Raccourci envoyé
- **WHEN** un geste mappé sur Commande + Maj + 4 est effectué
- **THEN** l'application au premier plan reçoit cette combinaison une fois

#### Scenario: Raccourci de l'app elle-même
- **WHEN** le raccourci enregistré est l'un des raccourcis globaux de Vorssaint
- **THEN** la fonction correspondante de Vorssaint se déclenche, comme si la touche avait été pressée

### Requirement: Navigation Spaces et vues d'ensemble
Les actions Space précédent, Space suivant, Mission Control et App Exposé DOIVENT utiliser le raccourci clavier système que l'utilisateur a configuré pour ces fonctions, afin de respecter ses remappages.

#### Scenario: Space suivant
- **WHEN** un geste mappé sur Space suivant est effectué
- **THEN** le bureau passe au Space de droite, exactement comme le raccourci système

#### Scenario: Raccourci système désactivé
- **WHEN** l'utilisateur a désactivé le raccourci système de Mission Control
- **THEN** l'action ne fait rien et les réglages indiquent que le raccourci système est requis

### Requirement: Disposition de fenêtre
L'action disposition DOIT appliquer la disposition choisie à la fenêtre au premier plan avec le même résultat que la feature Disposition des fenêtres.

#### Scenario: Moitié gauche
- **WHEN** un geste mappé sur « moitié gauche » est effectué
- **THEN** la fenêtre au premier plan occupe la moitié gauche de son écran

### Requirement: Outils et bascules Vorssaint
L'action outil DOIT ouvrir ou basculer l'outil choisi (y compris la Command Bar) et l'action bascule rapide DOIT appliquer la bascule choisie, avec le même effet que depuis le menu radial. Le sélecteur d'apps n'est pas un outil proposé : sa session dépend d'un raccourci maintenu.

#### Scenario: Presse-papiers
- **WHEN** un geste mappé sur l'historique du presse-papiers est effectué
- **THEN** la fenêtre d'historique s'ouvre, ou se ferme si elle était ouverte

#### Scenario: Outil désinstallé
- **WHEN** l'outil mappé n'est plus installé
- **THEN** le geste ne fait rien et les réglages montrent l'action comme indisponible avec un bouton vers Features

### Requirement: Média, application et URL
L'action média DOIT envoyer la touche média choisie ; l'action application DOIT lancer ou activer l'app choisie ; l'action URL DOIT ouvrir le lien dans le navigateur par défaut.

#### Scenario: Lecture pause
- **WHEN** un geste mappé sur lecture/pause est effectué
- **THEN** le lecteur actif bascule entre lecture et pause

#### Scenario: Application déplacée
- **WHEN** l'app mappée n'existe plus à son chemin
- **THEN** le geste ne fait rien et les réglages montrent l'action comme indisponible

### Requirement: Une exécution par geste
Chaque geste reconnu DOIT exécuter son action une seule fois, sur le fil principal, sans retarder le traitement des événements d'entrée.

#### Scenario: Geste long
- **WHEN** un balayage continue bien au-delà du seuil avant la levée des doigts
- **THEN** l'action est exécutée une seule fois
