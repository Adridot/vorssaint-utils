# Design

## Context

Voir proposal.md pour la motivation et `openspec/explorations/trackpad-gestures.md` pour l'exploration complète.

État actuel qui contraint l'approche :

- `MiddleClickService` (Services/MiddleClick) possède l'unique pont vers le framework privé MultitouchSupport : `dlopen` du framework, `MTDeviceCreateList`, `MTRegisterContactFrameCallback`, `MTDeviceStart/Stop`, hotplug IOKit sur `AppleMultitouchDevice`, reprise au réveil. Il lit seulement x et y de chaque touche (offsets 32 et 36, pas de 96 octets) et calcule centroïde et écartement. Ses seuils de tap (`MiddleClickSupport`) sont calibrés et testés.
- Le framework n'accepte qu'un callback par device. Le tap 4 doigts du menu radial passe déjà par ce service via `MiddleClickSupport.radialMenuTapFingers` et `RadialMenuService.toggleFromTrackpad()`.
- Le framework est en lecture seule : il observe les contacts, il ne peut pas empêcher macOS d'agir sur ses propres gestes.
- `RadialMenuItem` (Services/RadialMenu/RadialMenuSupport.swift) est le catalogue d'actions Codable le plus complet ; ses exécuteurs `run(_:)`, `postShortcut`, `postAuxKey` sont privés dans `RadialMenuService`. `SpaceWindowBridge.spaceShortcut/overviewShortcut/pressSpaceShortcut` rejouent le raccourci symbolique système.
- Conventions : `AppFeature` avec switches exhaustifs, `FeatureRuntime.bindings`, `SessionActivitySupport.tapShouldRun`, `MouseAppExceptions` par scope, chaînes en structs Swift par langue (15), tests dans un binaire maison qui ne compile que les fichiers `*Support.swift` et `*Strings.swift` listés dans `build.sh`.
- Cet environnement de travail est Linux : aucune compilation Swift ni essai sur trackpad n'est possible ici.

## Goals / Non-Goals

**Goals:**
- Une seule source de contacts partagée par le clic du milieu, le menu radial et les gestes.
- Une reconnaissance pure et testée sans matériel, qui n'utilise que centroïde et écartement.
- Un exécuteur d'actions partagé pour que menu radial et gestes proposent le même catalogue.
- Zéro régression observable sur le clic du milieu et le tap du menu radial.
- Dégradation silencieuse si le framework privé change.

**Non-Goals:**
- Supprimer ou intercepter les gestes système.
- Lire les identifiants de doigt, l'angle ou la pression du struct de touche.
- Gestes continus (répétition pendant le mouvement), modificateurs requis, mappages par application, rotation, tip-tap, coins, scripts.
- Un dispatcher d'actions global pour toute l'app ; seul ce qui sert au menu radial et aux gestes est remonté.

## Decisions

### D1. Extraire un `TrackpadTouchService` partagé
Le pont `Multitouch` et la gestion des devices sortent de `MiddleClickService` vers `Services/Trackpad/TrackpadTouchService.swift`. Il expose `subscribe(_:) -> token` et distribue chaque frame (`TouchFrame { count, centroid, spread, uptime, positionsReadable }`) à ses abonnés sur le thread multitouch. Il démarre quand au moins un abonné est présent et s'arrête au dernier désabonnement.

Alternatives : enregistrer un second callback (impossible, un seul par device) ; faire de `TrackpadGestureService` un sous-abonné du clic du milieu (couplage inversé, le clic du milieu désactivé couperait les gestes). Rejetées.

### D2. Reconnaissance pure dans `TrackpadGestureSupport`
`Services/Trackpad/TrackpadGestureSupport.swift` contient `GestureKind` (tap, swipe(direction), pinch(in/out)), `TrackpadGesture { kind, fingers }` identifiable par une clé stable (ex. `swipe-left-4`), et une `Recognizer` à état : `idle → candidate(fingers, start, maxMovement, maxSpreadDelta, flags) → fired | cancelled` jugée quand `count == 0`. Les seuils vivent ensemble, nommés : `tapMaxDuration` 0,35 s, `tapMovementLimit` 0,03, `swipeMovementThreshold` 0,08 (normalisé), `pinchSpreadThreshold` 0,05, `swipeOverPinchDominance` 2, `axisDominance` 1,5, `keyboardIdle` 0,5 s, `maxGestureDuration` 1,5 s.

Calibré sur Mac (MacBook Pro 14 pouces M1 Pro, trackpad intégré, macOS 27.0.1, environ 140 contacts) : les valeurs de départ 0,12 et 0,08 ne reconnaissaient que 70 balayages sur 89 et 28 pincements sur 37. Les balayages mesurés vont de 0,10 à 0,29 (médiane 0,15), les pincements de 0,056 à 0,165 (médiane 0,10), les taps restent sous 0,03. Deux seuils indépendants ne suffisent pas : une main qui glisse change d'écartement jusqu'à 0,38 fois son déplacement, et un pincement entraîne le centre jusqu'à 0,9 fois son changement d'écartement. La règle « les deux seuils dépassés, rien ne part » devient donc une règle de dominance : balayage si le déplacement vaut au moins deux fois le changement d'écartement, pincement si le changement d'écartement vaut au moins le déplacement, rien entre les deux. Sur les mêmes contacts : 85 balayages sur 89, les 34 pincements à 4 et 5 doigts, aucune confusion entre les deux. Non mesuré : balayages à 5 doigts, Magic Trackpad.

Le tap réutilise exactement la logique de `MiddleClickSupport.tapShouldFire`. Le clic du milieu garde son propre `actionForClick` (il porte sur le clic physique, pas un geste).

Alternative : suivre chaque doigt par identifiant pour la rotation et le tip-tap. Rejetée pour la v1 : offsets non vérifiés sur macOS 14 à 26.

### D3. Un seul propriétaire des taps : le reconnaisseur de gestes
`TrackpadGestureService` devient l'unique reconnaisseur de taps. Le tap du clic du milieu et le tap du menu radial sont des réservations : `TrackpadGestureSupport.reservations(middleClickTapFingers:, radialMenuTapFingers:)` marque `tap-3` ou `tap-4` comme pris, et le service route un tap réservé vers `MiddleClickService.postMiddleTap()` ou `RadialMenuService.toggleFromTrackpad()` au lieu du mappage. Les réglages des gestes affichent la réservation. `MiddleClickService` ne garde que le clic physique (tap d'événements et `actionForClick`).

Alternative : laisser le tap dans `MiddleClickService` avec une règle de priorité entre deux reconnaisseurs. Rejetée : deux machines à états sur les mêmes frames produisent des doubles déclenchements.

Conséquence : le reconnaisseur tourne dès que l'un des trois consommateurs le demande, même si la feature gestes n'est pas installée. Les mappages ne s'appliquent que si `AppFeature.trackpadGestures.isAvailable` et `trackpadGesturesEnabled`.

### D4. Actions : réutiliser `RadialMenuItem`, ajouter la navigation
Le mappage persiste en `Data` JSON sous `DefaultsKey.trackpadGestureMappings` : `[gestureKey: TriggerAction]` où `TriggerAction` est un enum Codable `case item(RadialMenuItem)` et `case navigation(SpaceNavigation)` avec `spaceLeft, spaceRight, missionControl, appExpose`. Le sous-menu (`.submenu`) est exclu des gestes par sanitisation.

Les exécuteurs `run(_ item:)`, `run(_ tool:)`, `run(_ toggle:)`, `run(_ layout:)`, `postShortcut`, `postAuxKey`, `postWhenModifiersReleased` quittent `RadialMenuService` pour `Services/TriggerActionRunner.swift` (enum statique, main thread). `RadialMenuService` l'appelle. La navigation passe par `SpaceWindowBridge`. `RadialMenuTool` gagne `commandBar`, ce qui bénéficie aussi au menu radial.

Révisé à l'implémentation : le sélecteur d'apps n'est pas ajouté. Sa session est liée à un raccourci maintenu et se valide au relâchement des modificateurs ; l'ouvrir depuis un geste demande un mode de session persistant dans `AppSwitcher`, qui dépasse une simple entrée publique. Décision du porteur du projet : le sélecteur d'apps sort du périmètre de ce change ; il pourra faire l'objet d'un change séparé qui ajoute un mode de session persistant.

Révisé après essai sur Mac : `TriggerAction` gagne `case backForward(MouseNavigationDirection)` (Précédent, Suivant), demandé dès le premier essai pour naviguer dans un navigateur. `MouseNavigationService.navigate(_:)` réutilise la recherche de menu des boutons latéraux (commande de l'app au premier plan, touche résolue par `MouseNavigationKeys` pour les claviers qui ne tapent pas les crochets). Quand les menus répondent en entier sans la commande (famille Firefox, dont Zen), le service poste le bouton latéral 3 ou 4 à la fenêtre sous le pointeur, décision pure `MouseNavigationSupport.sendsSideButton`. Un raccourci ⌘[ mappé à la main ne convenait pas : il n'est pas tapable sur un clavier AZERTY. Les libellés viennent de `SettingsNavigationStrings`, déjà traduits.

Alternative : un nouvel enum d'actions propre aux gestes. Rejetée : duplication du sélecteur d'action et de la validation déjà écrits pour le menu radial.

### D5. Conflits système lus dans les préférences, jamais bloqués
`TrackpadGestureSupport.systemClaims(prefs:)` est une fonction pure qui prend un dictionnaire de valeurs de préférences et renvoie `[gestureKey: SystemClaim]` avec le chemin Réglages Système. Le service lit les domaines `com.apple.AppleMultitouchTrackpad`, `com.apple.driver.AppleBluetoothMultitouch.trackpad` et `com.apple.dock` par `CFPreferencesCopyAppValue`, sur le thread principal, en cache 2 s comme le fait déjà le clic du milieu pour `TrackpadThreeFingerDrag`. Les clés exactes (`TrackpadThreeFingerHorizSwipeGesture`, `TrackpadFourFingerHorizSwipeGesture`, `TrackpadThreeFingerVertSwipeGesture`, `TrackpadFourFingerVertSwipeGesture`, `TrackpadFourFingerPinchGesture`, `TrackpadFiveFingerPinchGesture`, `TrackpadThreeFingerTapGesture`, `TrackpadThreeFingerDrag`, `showMissionControlGestureEnabled`, `showAppExposeGestureEnabled`, `showDesktopGestureEnabled`, `showLaunchpadGestureEnabled`) sont à confirmer sur Mac avec `defaults read` avant d'écrire les tests.

Un geste pris reste mappable mais la ligne est marquée ; l'action s'exécute quand même si l'utilisateur insiste. Alternative : refuser le mappage. Rejetée, certains utilisateurs veulent les deux effets.

### D6. Exclusions par application via `MouseExceptionScope.trackpadGestures`
Nouveau scope, clé `trackpadGestureExceptions`, UI `MouseExceptionsList(scope:)`. Le test se fait sur l'app au premier plan (`excludesActionTarget`) au moment de la levée des doigts, sur le thread principal.

### D7. Threads et verrous
Frames sur le thread multitouch, machine à états sous un `NSLock` dans le service, décision émise puis `DispatchQueue.main.async` pour les exclusions, les réservations et l'exécution. Aucune lecture de préférence ni d'UI sur le thread multitouch.

Révisé à l'implémentation : le service des gestes n'ouvre aucun tap d'événements. À la levée des doigts, il lit `CGEventSource.secondsSinceLastEventType(.combinedSessionState, …)` pour `keyDown` (frappe récente) et pour `leftMouseDown`, `rightMouseDown`, `otherMouseDown` (clic pendant le contact : un clic plus récent que la durée du contact annule le geste). C'est la même source que le contrôle de frappe existant du clic du milieu, sans tap supplémentaire à maintenir ni à réarmer. À confirmer sur Mac : qu'un clic physique transformé en clic du milieu est bien vu par cette lecture.

Révisé après essais sur Mac, en deux temps. Constat : le registre de la dernière frappe compte aussi le raccourci posté par l'action du geste précédent ; sur une série rapide de balayages mappés sur Espace précédent ou suivant, 10 gestes sur 25 étaient rejetés comme frappe récente. Première tentative, fausse : lire `.hidSystemState` en supposant qu'il ne contient que les touches matérielles. Mesuré : les deux registres donnent la même valeur après un raccourci posté par l'app (`post(tap: .cghidEventTap)`). Correction retenue : `OwnKeyEvent.notePosted()` date les touches que l'exécuteur poste, et la fonction pure `TrackpadGestureSupport.secondsSinceTyping` ignore une frappe qui coïncide (à 0,1 s près) avec ce dernier envoi.

Second constat des mêmes logs : `postWhenModifiersReleased` attendait jusqu'à 78 tours (1,2 s) avant de poster, sans aucune touche tenue. Le Ctrl+flèche posté par le geste précédent reste lu comme modificateur tenu par `flagsState(.combinedSessionState)`, et deux actions en attente partaient au même instant. Un geste n'a pas d'accord clavier à laisser relâcher : `TriggerActionRunner.Context.waitsForModifierRelease` vaut `false` pour les gestes et garde `true` pour le menu radial, dont le comportement ne change pas. Lire l'état physique des touches par `CGEventSource.keyState` a été écarté : ce dépôt documente qu'il bloque l'app depuis le fil principal (`SuperKeyService.readPhysicalKeyDown`).

### D8. Réglages
Carte « Gestes du trackpad » dans `MouseSettings` (ancre `.trackpadGestures`), avec un interrupteur, une liste de lignes par geste groupées par nombre de doigts (icône, nom, menu d'action, note « pris par macOS / clic du milieu / menu radial / action indisponible »), l'avertissement trackpad absent, puis `MouseExceptionsList`. Si la liste dépasse la carte, bascule vers une page dédiée ; décision prise sur capture d'écran, sans impact sur les specs.

Révisé à l'implémentation : plutôt qu'un nouveau `TriggerActionPicker`, l'éditeur d'action existant du menu radial (`RadialItemEditor`) passe de `private` à interne et sert tel quel en feuille, avec `allowsSubmenu: false`. Le menu de chaque ligne propose « Aucune action », les quatre navigations et « Autre action », qui ouvre cet éditeur. Les deux features partagent ainsi le même éditeur sans toucher à son code.

### D9. Tests
Tout ce qui décide est pur : `TrackpadGestureSupport` (reconnaissance, réservations, claims système, encodage/décodage des mappages), `TriggerAction` (sanitisation), chaînes. Les suites vont dans `Tests/PointerInputFeatureTests.swift` (groupe `pointer-input`) et `Tests/LocalizationTests.swift` (fabrique ajoutée). Les fichiers sont ajoutés à `TEST_SOURCES` dans `build.sh`. Les comptes du catalogue dans `Tests/FeatureCatalogTests.swift` passent à 78.

## Risks / Trade-offs

- [Le struct de touche change sur une version de macOS] → lecture limitée à x/y avec vérification de plage, échec fermé par contact, feature inactive sans plantage ; `--selftest` ajoute un avertissement si les symboles manquent.
- [Seuils de balayage et de pincement mal calibrés] → constantes nommées au même endroit, tests sur des trajectoires synthétiques, calibration sur Mac avant la PR avec les valeurs notées dans la PR.
- [Double déclenchement avec un geste macOS] → marquage dans les réglages et chemin pour libérer le geste ; recommandation des gestes libres (5 doigts, tap 4 doigts) dans la description.
- [Régression du clic du milieu lors de l'extraction] → le tap d'événements et `actionForClick` ne bougent pas ; tests existants de `MiddleClickSupport` conservés ; vérification manuelle du clic 3 doigts, du tap 3/4 doigts et du tap menu radial avant et après.
- [Le reconnaisseur tourne pour le menu radial même sans la feature gestes] → coût identique à aujourd'hui (le clic du milieu lisait déjà les frames) ; les mappages sont ignorés tant que la feature n'est pas installée.
- [Aucune vérification possible dans cet environnement] → la PR devra être ouverte en brouillon avec la liste des vérifications restantes, puis complétée après essais sur Mac (macOS 14, 15, 26, trackpad intégré et Magic Trackpad).
- [Le mainteneur attend une discussion préalable pour un nouveau sous-système] → ouvrir ou rejoindre vorssaint/vorssaint-utils#2109 avant d'implémenter ; la PR vorssaint/vorssaint-utils#2660 (Spaces au trackpad) est à relire pour éviter deux moteurs.

## Migration Plan

- Aucune migration de données : nouvelles clés seulement. Les clés `middleClickTapFingers` et `RadialMenuProfile.trackpadTap` gardent leur sens et deviennent des réservations.
- Déploiement : feature opt-in, invisible pour qui ne l'installe pas.
- Retour arrière : désinstaller la feature. Les mappages restent en préférences et sont ignorés.

## Open Questions


- Carte dans la page Souris ou page dédiée : tranché à l'implémentation sur capture d'écran (D8), sans effet sur specs ni tâches.
- Valeurs finales des seuils de balayage et de pincement : calibrées sur Mac (D2).
- Noms exacts des clés de préférences système pour les claims (D5) : confirmés avec `defaults read` sur Mac avant d'écrire les tests correspondants.
