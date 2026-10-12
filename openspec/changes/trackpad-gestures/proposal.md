# Proposal

## Why

Les utilisateurs de BetterTouchTool ou Multitouch veulent déclencher des actions avec des gestes à 3, 4 ou 5 doigts sur le trackpad, et la discussion amont vorssaint/vorssaint-utils#2109 réclame ce remappage. Vorssaint lit déjà les contacts bruts du trackpad pour le clic du milieu et le tap du menu radial, mais ne reconnaît qu'un tap immobile : le socle existe, il manque le moteur de gestes et le mappage vers des actions.

## What Changes

- Nouvelle feature opt-in **Gestes du trackpad** (`trackpadGestures`) dans le groupe Souris et clavier, installable depuis la page Features, qui requiert Accessibilité.
- Reconnaissance de gestes à partir des contacts bruts : tap 3/4/5 doigts, balayage 3/4/5 doigts (gauche, droite, haut, bas), pincement 4/5 doigts (fermer, ouvrir). Une seule action par contact, jugée à la levée des doigts.
- Mappage configurable geste → action, avec le même catalogue d'actions que le menu radial (raccourci clavier, outil Vorssaint, bascule rapide, disposition de fenêtre, touche média, ouvrir une app ou une URL) plus les actions Spaces : Space précédent ou suivant, Mission Control, App Exposé.
- Détection des gestes déjà pris par macOS (balayages 3 et 4 doigts, pincements système, tap 3 doigts Rechercher, glisser à trois doigts) à partir des préférences système, affichée dans les réglages avec le chemin pour libérer le geste. Aucune tentative de supprimer un geste système.
- Liste d'apps exclues (nouveau scope d'exceptions souris), réglages sauvegardés et exportés avec les autres préférences, 15 langues.
- Refactor interne : extraction du pont MultitouchSupport du service de clic du milieu vers un service de contacts partagé, et des exécuteurs d'actions du menu radial vers un exécuteur partagé. Le clic du milieu et le tap 4 doigts du menu radial gardent exactement leur comportement.

Hors périmètre de ce change : mappages par application, modificateurs requis, rotation, tip-tap, coins, scripts shell, retour haptique, répétition continue pendant le geste.

## Capabilities

### New Capabilities
- `trackpad-gestures` : reconnaissance des gestes multi-doigts sur le trackpad, configuration des mappages, exclusions par application, signalement des conflits avec les gestes macOS et de l'absence de trackpad, cycle de vie lié à la feature et à la permission Accessibilité.
- `trackpad-gesture-actions` : les actions qu'un geste mappé peut déclencher et leur effet observable (raccourci clavier, navigation Spaces et vues d'ensemble, disposition de fenêtre, outils Vorssaint, bascules rapides, médias, ouverture d'app ou d'URL), y compris le comportement quand l'action n'est plus disponible.

### Modified Capabilities

Aucune : le dépôt n'a pas encore de specs OpenSpec. Le clic du milieu et le tap du menu radial ne changent pas de comportement observable.

## Impact

- `Sources/Vorssaint/Core` : nouveau cas `AppFeature.trackpadGestures` et ses switches exhaustifs (groupe, symbole, clés, permissions, `installedByDefault` false, destination des réglages), nouvelles `DefaultsKey`, nouveau catalogue de chaînes 15 langues, descriptions du hub, mots-clés de recherche.
- `Sources/Vorssaint/Services` : nouveau `TrackpadTouchService` (pont MultitouchSupport partagé, hotplug), `TrackpadGestureSupport` (reconnaissance pure), `TrackpadGestureService` (mappage, exécution), `TriggerActionRunner` partagé ; `MiddleClickService` et `RadialMenuService` deviennent des abonnés.
- `Sources/Vorssaint/App` : binding dans `FeatureRuntime`, sink Accessibilité et `suspend()` à la fermeture dans `AppDelegate`, suspensions dans `SelfUninstall` et `CleaningModeManager`.
- `Sources/Vorssaint/UI/Settings` : carte Gestes du trackpad dans la page Souris avec ancre, légende et mots-clés ; `MouseExceptionsList` avec le nouveau scope.
- `build.sh` : fichiers `*Support.swift` et `*Strings.swift` ajoutés à `TEST_SOURCES`. Tests : catalogue (77 → 78 features), localisation, reconnaissance des gestes, mappage, exceptions.
- `docs/PERMISSIONS.md` : ligne pour la nouvelle feature. `CHANGELOG.md` et version laissés au mainteneur.
- Dépendance : framework privé MultitouchSupport déjà utilisé, chargé par `dlopen`, aucune nouvelle permission. Vérification obligatoire sur Mac réel (macOS 14, 15 et 26, trackpad intégré et Magic Trackpad) : cet environnement ne compile pas Swift.
