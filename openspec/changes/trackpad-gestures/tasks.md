# Tasks

Toutes les tâches de code se vérifient sur un Mac avec `./build.sh --dev`, `./build/VorssaintDeveloper --selftest` et `./build.sh --test` (ou `--test-suite=NAME`). Le code a été écrit sous Linux, puis vérifié sur un MacBook Pro 14 pouces M1 Pro, macOS 27.0.1, trackpad intégré. Une tâche n'est cochée que si sa vérification a réellement été faite ; la note d'état dit ce qui a été observé et ce qui ne l'a pas été.

## 1. Service de contacts partagé

- [x] 1.1 Créer `Sources/Vorssaint/Services/Trackpad/TrackpadTouchService.swift` en déplaçant le pont `Multitouch` (dlopen, symboles, `touchGeometry`), la liste des devices, le hotplug IOKit et la reprise au réveil depuis `MiddleClickService` ; exposer `subscribe/unsubscribe`, `available`, `deviceMissing` publié ; vérifier que `./build.sh --dev` compile sans avertissement.
  - État : vérifié : `./build.sh --dev` compile du premier coup, sans erreur ni avertissement. Le hotplug et la reprise au réveil n'ont pas été exercés.
- [ ] 1.2 Faire de `MiddleClickService` un abonné qui ne garde que le clic physique (tap d'événements, `actionForClick`, `fingerCount`, `lastFrameUptime`, `fingerCountSince`) ; vérifier sur Mac que le clic 3 doigts (4 avec glisser à trois doigts) donne toujours un clic du milieu et que la désactivation stoppe la lecture.
  - État : le clic physique à 3 doigts donne bien un clic du milieu sur Mac (vu par l'utilisateur, et le contact est rejeté comme geste). Non vérifié : le clic à 4 doigts avec le glisser à trois doigts actif, et l'arrêt de la lecture à la désactivation.
- [x] 1.3 Étendre `Tests/RepositoryFeatureTests.swift` si le déplacement du `tapCreate` change les fichiers inspectés, et vérifier que `./build.sh --test-suite=repository` passe.
  - État : vérifié : `./build.sh --test-suite=repository` passe.
- [ ] 1.4 Ajouter dans `Support/SelfTest.swift` un avertissement (non bloquant) quand `TrackpadTouchService.available` est faux ; vérifier que `--selftest` passe sur un Mac avec et sans trackpad.
  - État : `--selftest` passe sur un Mac avec trackpad. Le cas sans trackpad n'est pas testable sur un MacBook.

## 2. Reconnaissance pure des gestes

- [x] 2.1 Créer `Services/Trackpad/TrackpadGestureSupport.swift` : `GestureKind`, `TrackpadGesture` avec clé stable, `Recognizer` à état alimentée par des frames synthétiques (`count, centroid, spread, uptime, buttonPressed, secondsSinceLastKeyDown`), constantes nommées (D2) ; ajouter le fichier à `TEST_SOURCES` dans `build.sh`.
  - État : vérifié : compilé avec le SDK macOS et couvert par la suite `pointer-input`.
- [x] 2.2 Écrire les tests dans `Tests/PointerInputFeatureTests.swift` (groupe `pointer-input`) : tap 3/4/5 doigts fire, balayage dans les quatre directions avec dominance d'axe, pincement fermer/ouvrir, annulation par doigt supplémentaire, clic, frappe récente, position illisible, durée trop longue, contact ambigu ; vérifier que `./build.sh --test-suite=pointer-input` échoue sans la reconnaissance et passe avec.
  - État : vérifié sur Mac : la suite `pointer-input` passe (1 085 vérifications) et échoue pour chacune de quatre mutations volontaires (axe vertical inversé, seuil de balayage remis à 0,12, règle de dominance retirée, frappe synthétique comptée).
- [x] 2.3 Ajouter `reservations(middleClickTapFingers:radialMenuTapFingers:)` et ses tests (tap-3 et tap-4 réservés, aucun si tout est désactivé), en reproduisant la règle de `MiddleClickSupport.radialMenuTapFingers` ; vérifier avec la même suite.
  - État : vérifié par la suite et sur Mac : tap à 3 doigts routé vers le clic du milieu, tap à 4 doigts vers le menu radial, et le clic du milieu l'emporte quand les deux veulent le tap à 4 doigts.
- [ ] 2.4 Ajouter `systemClaims(prefs:)` pur avec le chemin Réglages Système par geste, après confirmation des clés par `defaults read com.apple.AppleMultitouchTrackpad` et `defaults read com.apple.dock` sur Mac ; tests pour balayage 3 doigts Spaces, 4 doigts, pincements, tap 3 doigts Rechercher, glisser à trois doigts ; noter les clés confirmées dans la PR.
  - État : confirmés par `defaults read` et par bascule dans Réglages Système : les huit clés `Trackpad…` (mêmes valeurs dans les deux domaines), `showMissionControlGestureEnabled` (1 puis 0), `showAppExposeGestureEnabled`, `showDesktopGestureEnabled` (absente puis 0). Corrigé à l'essai : les balayages à cinq doigts suivent les réglages à quatre doigts. Non confirmés : la clé du pincement, qui s'appelle « Apps » sur macOS 27 et n'a pas pu être désactivé (`showLaunchpadGestureEnabled` n'y est jamais écrite), la valeur de `TrackpadThreeFingerTapGesture` quand Rechercher est actif, `TrackpadThreeFingerDrag` actif.

## 3. Exécuteur d'actions partagé

- [ ] 3.1 Créer `Services/TriggerActionRunner.swift` en déplaçant `run(_:)`, `run(_ tool:)`, `run(_ toggle:)`, `run(_ layout:)`, `postShortcut`, `postAuxKey`, `postWhenModifiersReleased` depuis `RadialMenuService`, qui l'appelle désormais ; vérifier sur Mac que chaque famille d'actions du menu radial fonctionne comme avant (app, fichier, URL, raccourci, outil, bascule, disposition, média).
  - État : depuis un geste, vérifié sur Mac : app, URL, raccourci, outil, disposition, média. Non vérifié : les mêmes actions depuis la roue du menu radial (elle s'ouvre, aucune tranche n'a été lancée), fichier et bascule rapide.
- [x] 3.2 Ajouter `TriggerAction` Codable (`item(RadialMenuItem)`, `navigation(SpaceNavigation)`) dans `Services/Trackpad/TrackpadGestureSupport.swift`, avec encodage/décodage tolérant des mappages `[gestureKey: TriggerAction]`, sanitisation (pas de `.submenu`, payload valide via `RadialMenuSupport.isValidPayload`) et tests de round-trip et d'entrées invalides.
  - État : vérifié : les tests de mappage passent sur Mac contre les vrais types.
- [x] 3.3 Brancher la navigation sur `SpaceWindowBridge.spaceShortcut/overviewShortcut/pressSpaceShortcut` et exposer `isAvailable` par action (raccourci système actif) ; vérifier sur Mac Space gauche/droite, Mission Control, App Exposé, et le cas raccourci système désactivé.
  - État : vérifié sur Mac : Espace précédent et suivant, Mission Control, App Exposé, et le raccourci système Mission Control désactivé (note affichée, geste sans effet).
- [ ] 3.4 Ajouter `commandBar` à `RadialMenuTool` ; mettre à jour les tests du menu radial qui épinglent la liste des outils ; vérifier sur Mac depuis le menu radial et depuis un geste.
  - État : la Command Bar s'ouvre depuis un geste. Non vérifié depuis le menu radial.

- [x] 3.5 Ajouter `TriggerAction.backForward` (Précédent, Suivant) : `MouseNavigationService.navigate(_:)`, décision pure `MouseNavigationSupport.sendsSideButton` et ses tests, entrées du menu d'action ; vérifier sur Mac dans Safari ou le Finder (commande de menu) et dans un navigateur de la famille Firefox (bouton latéral).
  - État : vérifié sur Mac : Zen (bouton latéral, les menus n'ont pas la commande), Finder et Claude Desktop (commande de menu). Safari non essayé.

## 4. Feature, préférences, cycle de vie

- [x] 4.1 Ajouter `AppFeature.trackpadGestures` (groupe `.mouseKeyboard`, symbole `hand.draw`, `enabledKeys` `[trackpadGesturesEnabled]`, permissions `[.accessibility]`, `installedByDefault` false, `energyProfile .mouse`, destination `.mouse` avec ancre `.trackpadGestures`, titre et description du hub) et mettre à jour `Tests/FeatureCatalogTests.swift` (78 features, ordre des raw values, ensemble installé par défaut) et `Tests/UpdateFeatureTests.swift` ; vérifier `./build.sh --test-suite=features`.
  - État : vérifié : suite `features` verte, et la feature s'installe depuis le hub.
- [ ] 4.2 Ajouter `DefaultsKey.trackpadGesturesEnabled` (false), `trackpadGestureMappings` (Data, dans `unregisteredPreferenceKeys`), `trackpadGestureExceptions` ([String]) et `MouseExceptionScope.trackpadGestures` ; vérifier `Tests/PreferencesFeatureTests` et le round-trip export/import avec `SettingsBackupSupport`.
  - État : clés, sauvegarde et scope d'exception en place ; la suite `preferences` passe et vérifie désormais que les trois clés des gestes font partie de l'export. Non vérifié : un export puis un import à la main, et le tri des entrées invalides à l'import.
- [x] 4.3 Créer `Services/Trackpad/TrackpadGestureService.swift` : abonné de `TrackpadTouchService`, `Recognizer` sous verrou, routage des taps réservés vers `MiddleClickService.postMiddleTap()` et `RadialMenuService.toggleFromTrackpad()`, exclusions, exécution via `TriggerActionRunner` sur le fil principal, tap d'écoute `listenOnly` des clics sur `PointerTapRunLoop` avec re-armement `tapDisabledBy*`, `syncWithPreferences`, `suspend`, cache 2 s des claims système ; vérifier sur Mac qu'un geste mappé agit une fois et qu'un geste non mappé ne fait rien.
  - État : vérifié sur Mac : un geste mappé lance son action une fois, un geste non mappé ne fait rien. Écart au design D7 maintenu : aucun tap d'écoute.
- [x] 4.4 Retirer la reconnaissance de tap de `MiddleClickService` et `RadialMenuService.syncWithPreferences` appelle `TrackpadGestureService.shared.syncWithPreferences()` ; vérifier sur Mac le tap 3 doigts clic du milieu, le tap 4 doigts menu radial et la priorité entre les deux, feature gestes installée ou non.
  - État : vérifié sur Mac, feature gestes installée puis désinstallée : tap à 3 doigts en clic du milieu, tap à 4 doigts en menu radial, priorité du clic du milieu sur le tap à 4 doigts.
- [ ] 4.5 Brancher le cycle de vie : `FeatureRuntime.bindings`, sink Accessibilité et `suspend()` à la fermeture dans `AppDelegate`, `SelfUninstall.suspendInputInterceptors`, `CleaningModeManager` ; vérifier sur Mac que l'octroi d'Accessibilité démarre les gestes sans redémarrage et que le mode nettoyage les coupe.
  - État : mode nettoyage vérifié sur Mac après correction (aucun contact lu entre l'entrée et la sortie). Désinstallation puis réinstallation vérifiées, mappages conservés. Non vérifié : l'octroi d'Accessibilité après activation de la feature.

- [x] 4.6 Ne pas compter comme frappe le raccourci posté par l'action du geste précédent (`OwnKeyEvent.notePosted`, `TrackpadGestureSupport.secondsSinceTyping`) et ne pas attendre le relâchement de modificateurs pour un geste (`Context.waitsForModifierRelease`) ; vérifier sur Mac qu'une série rapide de balayages mappés sur un raccourci passe en entier, sans retard, et qu'un contact juste après une vraie frappe reste rejeté.
  - État : vérifié sur Mac : 23 balayages rapides sur 23 exécutés, chacun posté en moins de 10 ms ; 8 contacts rejetés entre 0,20 et 0,48 s après une vraie frappe, accepté à 0,71 s. La première correction (`.hidSystemState`) était fausse et a été remplacée.

## 5. Chaînes et réglages

- [x] 5.1 Créer `Core/TrackpadGestureStrings.swift` (struct, 15 langues, résolveur dans `FeatureStrings`, noms de gestes, états pris par macOS / clic du milieu / menu radial, chemins Réglages Système, avertissements trackpad absent et raccourci système requis, descriptions du hub) ; ajouter le fichier à `TEST_SOURCES` et sa fabrique à `Tests/LocalizationTests.swift` ; vérifier `./build.sh --test-suite=localization`.
  - État : vérifié : suite `localization` verte (7 871 vérifications). Corrigé sur Mac : 18 noms d'onglets de Réglages Système dans 11 langues, alignés sur les tables de localisation d'Apple de macOS 27 (149 noms sur 150 identiques ; le volet Accessibilité en chinois simplifié est laissé tel quel). Noms de macOS 14, 15 et 26 non vérifiés.
- [ ] 5.2 Factoriser le sélecteur d'action du menu radial en `UI/Settings/TriggerActionPicker.swift` utilisable par les deux features ; vérifier sur Mac que l'éditeur du menu radial est inchangé.
  - État : écart au design D8 : `RadialItemEditor` est rendu interne et réutilisé tel quel. Non vérifié sur Mac : l'éditeur du menu radial n'a pas été rouvert.
- [ ] 5.3 Créer `UI/Settings/TrackpadGestureSettings.swift` (carte dans `MouseSettings`, interrupteur avec demande d'Accessibilité, grille des gestes avec badge d'état, picker, avertissements, `MouseExceptionsList(scope: .trackpadGestures)`), ancre `SettingsSectionAnchor.trackpadGestures`, légende et `isOn` dans `MouseSettings`, mots-clés dans `SettingsDirectory` ; décider carte ou page dédiée sur capture ; vérifier sur Mac le rendu dans les langues longues (de, ru, pt-BR).
  - État : carte vérifiée sur Mac en français (capture). Rendu dans les langues longues (de, ru, pt-BR) non vérifié.
- [x] 5.4 Afficher les actions indisponibles (outil désinstallé, app déplacée, raccourci système désactivé) grisées avec bouton vers Features ; vérifier sur Mac en désinstallant l'outil mappé.
  - État : vérifié sur Mac : outil désinstallé (note et bouton Fonctionnalités), app absente, raccourci système désactivé. Défaut corrigé à l'essai : le geste mappé sur une app absente faisait afficher l'alerte macOS.
- [x] 5.5 Ajouter la ligne de la feature dans `docs/PERMISSIONS.md` et vérifier le lien et le diff de la doc.

## 6. Vérification d'intégration sur Mac

- [x] 6.1 Build optimisé `./build.sh`, `./build/Vorssaint --selftest`, `./build.sh --test` complets sans avertissement.
  - État : vérifié sur le commit `fed5580b` : `./build.sh` sans erreur ni avertissement, `./build/Vorssaint --selftest` et `./build/VorssaintDeveloper --selftest` OK, `./build.sh --test` vert (116 013 vérifications, plus les tests de nettoyage des préférences, de désinstallation des Spaces et d'isolation de l'installation Developer).
- [ ] 6.2 Scénarios des specs rejoués sur trackpad intégré et Magic Trackpad : chaque geste reconnu, faux positifs rejetés (clic, frappe, repos), exclusion d'app, conflits marqués et libérés, branchement et débranchement, feature désinstallée puis réinstallée avec mappages conservés ; consigner Mac, macOS et ce qui reste non testé dans la PR.
  - État : rejoué sur le trackpad intégré (MacBook Pro 14 pouces M1 Pro, macOS 27.0.1, trackpad intégré) : chaque geste reconnu, faux positifs rejetés (clic, frappe, repos, six doigts), exclusion d'app, conflits marqués et libérés, désinstallation et réinstallation. Non fait : Magic Trackpad, branchement et débranchement, macOS 14, 15 et 26.
- [x] 6.3 Calibrer et figer les seuils de balayage et de pincement dans `TrackpadGestureSupport` avec les valeurs mesurées ; relancer la suite `pointer-input`.
  - État : fait : seuils 0,08 et 0,05, règle de dominance, sur environ 140 contacts ; suite `pointer-input` relancée.

## Workflow follow-up

- Ouvrir ou rejoindre vorssaint/vorssaint-utils#2109 pour valider la direction avec le mainteneur avant la PR, et relire vorssaint/vorssaint-utils#2660.
- Ouvrir la PR en brouillon `feat(trackpad): map multi-finger gestures to actions` sans les fichiers `openspec/` et `.claude/commands/opsx`, avec `Refs #2109`.
- Archiver le change OpenSpec une fois la PR fusionnée.
