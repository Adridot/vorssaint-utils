# Tasks

Toutes les tâches de code se vérifient sur un Mac avec `./build.sh --dev`, `./build/VorssaintDeveloper --selftest` et `./build.sh --test` (ou `--test-suite=NAME`). Cet environnement Linux ne compile pas Swift : une tâche n'est cochée qu'après vérification sur Mac.

## 1. Service de contacts partagé

- [ ] 1.1 Créer `Sources/Vorssaint/Services/Trackpad/TrackpadTouchService.swift` en déplaçant le pont `Multitouch` (dlopen, symboles, `touchGeometry`), la liste des devices, le hotplug IOKit et la reprise au réveil depuis `MiddleClickService` ; exposer `subscribe/unsubscribe`, `available`, `deviceMissing` publié ; vérifier que `./build.sh --dev` compile sans avertissement.
  - État : implémenté (`Services/Trackpad/TrackpadTouchService.swift`), parse OK ; compilation à vérifier sur Mac.
- [ ] 1.2 Faire de `MiddleClickService` un abonné qui ne garde que le clic physique (tap d'événements, `actionForClick`, `fingerCount`, `lastFrameUptime`, `fingerCountSince`) ; vérifier sur Mac que le clic 3 doigts (4 avec glisser à trois doigts) donne toujours un clic du milieu et que la désactivation stoppe la lecture.
  - État : implémenté ; `MiddleClickService` s'abonne et relaie l'avertissement trackpad absent. Essai sur Mac requis.
- [ ] 1.3 Étendre `Tests/RepositoryFeatureTests.swift` si le déplacement du `tapCreate` change les fichiers inspectés, et vérifier que `./build.sh --test-suite=repository` passe.
  - État : générateur et fixture `MiddleClickTrackpadContract` retargetés sur le nouveau service ; génération Python OK, compilation sur Mac requise.
- [ ] 1.4 Ajouter dans `Support/SelfTest.swift` un avertissement (non bloquant) quand `TrackpadTouchService.available` est faux ; vérifier que `--selftest` passe sur un Mac avec et sans trackpad.
  - État : avertissement ajouté dans `SelfTest.swift` ; `--selftest` à lancer sur Mac.

## 2. Reconnaissance pure des gestes

- [ ] 2.1 Créer `Services/Trackpad/TrackpadGestureSupport.swift` : `GestureKind`, `TrackpadGesture` avec clé stable, `Recognizer` à état alimentée par des frames synthétiques (`count, centroid, spread, uptime, buttonPressed, secondsSinceLastKeyDown`), constantes nommées (D2) ; ajouter le fichier à `TEST_SOURCES` dans `build.sh`.
  - État : implémenté, sans dépendance AppKit ; compilé et testé avec une toolchain Swift Linux (`-warnings-as-errors`).
- [ ] 2.2 Écrire les tests dans `Tests/PointerInputFeatureTests.swift` (groupe `pointer-input`) : tap 3/4/5 doigts fire, balayage dans les quatre directions avec dominance d'axe, pincement fermer/ouvrir, annulation par doigt supplémentaire, clic, frappe récente, position illisible, durée trop longue, contact ambigu ; vérifier que `./build.sh --test-suite=pointer-input` échoue sans la reconnaissance et passe avec.
  - État : `Tests/TrackpadGestureTests.swift`, 55 vérifications OK sous Linux ; mutations des règles détectées. Les anciens tests `tapShouldFire` y ont migré. Suite `pointer-input` à lancer sur Mac.
- [ ] 2.3 Ajouter `reservations(middleClickTapFingers:radialMenuTapFingers:)` et ses tests (tap-3 et tap-4 réservés, aucun si tout est désactivé), en reproduisant la règle de `MiddleClickSupport.radialMenuTapFingers` ; vérifier avec la même suite.
  - État : implémenté et testé sous Linux.
- [ ] 2.4 Ajouter `systemClaims(prefs:)` pur avec le chemin Réglages Système par geste, après confirmation des clés par `defaults read com.apple.AppleMultitouchTrackpad` et `defaults read com.apple.dock` sur Mac ; tests pour balayage 3 doigts Spaces, 4 doigts, pincements, tap 3 doigts Rechercher, glisser à trois doigts ; noter les clés confirmées dans la PR.
  - État : implémenté et testé sous Linux, mais les clés et valeurs par défaut de macOS restent à confirmer avec `defaults read` sur Mac.

## 3. Exécuteur d'actions partagé

- [ ] 3.1 Créer `Services/TriggerActionRunner.swift` en déplaçant `run(_:)`, `run(_ tool:)`, `run(_ toggle:)`, `run(_ layout:)`, `postShortcut`, `postAuxKey`, `postWhenModifiersReleased` depuis `RadialMenuService`, qui l'appelle désormais ; vérifier sur Mac que chaque famille d'actions du menu radial fonctionne comme avant (app, fichier, URL, raccourci, outil, bascule, disposition, média).
  - État : `Services/TriggerActionRunner.swift` ; le menu radial l'appelle. Essai de chaque famille d'actions sur Mac requis.
- [ ] 3.2 Ajouter `TriggerAction` Codable (`item(RadialMenuItem)`, `navigation(SpaceNavigation)`) dans `Services/Trackpad/TrackpadGestureSupport.swift`, avec encodage/décodage tolérant des mappages `[gestureKey: TriggerAction]`, sanitisation (pas de `.submenu`, payload valide via `RadialMenuSupport.isValidPayload`) et tests de round-trip et d'entrées invalides.
  - État : `TrackpadGestureMappingSupport.swift` + `Tests/TrackpadGestureMappingTests.swift`, 11 vérifications OK sous Linux contre des doublures des types AppKit ; à relancer sur Mac.
- [ ] 3.3 Brancher la navigation sur `SpaceWindowBridge.spaceShortcut/overviewShortcut/pressSpaceShortcut` et exposer `isAvailable` par action (raccourci système actif) ; vérifier sur Mac Space gauche/droite, Mission Control, App Exposé, et le cas raccourci système désactivé.
  - État : implémenté via `SpaceWindowBridge` ; essai sur Mac requis.
- [ ] 3.4 Ajouter `commandBar` à `RadialMenuTool` ; mettre à jour les tests du menu radial qui épinglent la liste des outils ; vérifier sur Mac depuis le menu radial et depuis un geste.
  - État : Command Bar ajoutée. Le sélecteur d'apps est retiré du périmètre (décision du porteur). Essai sur Mac requis.

## 4. Feature, préférences, cycle de vie

- [ ] 4.1 Ajouter `AppFeature.trackpadGestures` (groupe `.mouseKeyboard`, symbole `hand.draw`, `enabledKeys` `[trackpadGesturesEnabled]`, permissions `[.accessibility]`, `installedByDefault` false, `energyProfile .mouse`, destination `.mouse` avec ancre `.trackpadGestures`, titre et description du hub) et mettre à jour `Tests/FeatureCatalogTests.swift` (78 features, ordre des raw values, ensemble installé par défaut) et `Tests/UpdateFeatureTests.swift` ; vérifier `./build.sh --test-suite=features`.
  - État : catalogue et tests épinglés mis à jour ; compilation sur Mac requise.
- [ ] 4.2 Ajouter `DefaultsKey.trackpadGesturesEnabled` (false), `trackpadGestureMappings` (Data, dans `unregisteredPreferenceKeys`), `trackpadGestureExceptions` ([String]) et `MouseExceptionScope.trackpadGestures` ; vérifier `Tests/PreferencesFeatureTests` et le round-trip export/import avec `SettingsBackupSupport`.
  - État : clés, sauvegarde et scope d'exception ajoutés ; tests préférences à lancer sur Mac.
- [ ] 4.3 Créer `Services/Trackpad/TrackpadGestureService.swift` : abonné de `TrackpadTouchService`, `Recognizer` sous verrou, routage des taps réservés vers `MiddleClickService.postMiddleTap()` et `RadialMenuService.toggleFromTrackpad()`, exclusions, exécution via `TriggerActionRunner` sur le fil principal, tap d'écoute `listenOnly` des clics sur `PointerTapRunLoop` avec re-armement `tapDisabledBy*`, `syncWithPreferences`, `suspend`, cache 2 s des claims système ; vérifier sur Mac qu'un geste mappé agit une fois et qu'un geste non mappé ne fait rien.
  - État : implémenté. Écart au design D7 : aucun tap d'écoute, les clics et la frappe sont lus via `CGEventSource.secondsSinceLastEventType`. Essai sur Mac requis.
- [ ] 4.4 Retirer la reconnaissance de tap de `MiddleClickService` et `RadialMenuService.syncWithPreferences` appelle `TrackpadGestureService.shared.syncWithPreferences()` ; vérifier sur Mac le tap 3 doigts clic du milieu, le tap 4 doigts menu radial et la priorité entre les deux, feature gestes installée ou non.
  - État : reconnaissance de tap retirée de `MiddleClickService`, routage par réservations. Essai sur Mac requis (priorités clic du milieu, menu radial, gestes).
- [ ] 4.5 Brancher le cycle de vie : `FeatureRuntime.bindings`, sink Accessibilité et `suspend()` à la fermeture dans `AppDelegate`, `SelfUninstall.suspendInputInterceptors`, `CleaningModeManager` ; vérifier sur Mac que l'octroi d'Accessibilité démarre les gestes sans redémarrage et que le mode nettoyage les coupe.
  - État : branchements faits ; essai sur Mac requis.

## 5. Chaînes et réglages

- [ ] 5.1 Créer `Core/TrackpadGestureStrings.swift` (struct, 15 langues, résolveur dans `FeatureStrings`, noms de gestes, états pris par macOS / clic du milieu / menu radial, chemins Réglages Système, avertissements trackpad absent et raccourci système requis, descriptions du hub) ; ajouter le fichier à `TEST_SOURCES` et sa fabrique à `Tests/LocalizationTests.swift` ; vérifier `./build.sh --test-suite=localization`.
  - État : 15 langues, compilées sous Linux contre une doublure d'`AppLanguage` ; règles tiret cadratin, points de suspension et apostrophes vérifiées. Suite `localization` à lancer sur Mac.
- [ ] 5.2 Factoriser le sélecteur d'action du menu radial en `UI/Settings/TriggerActionPicker.swift` utilisable par les deux features ; vérifier sur Mac que l'éditeur du menu radial est inchangé.
  - État : écart au design D8 : `RadialItemEditor` est rendu interne et réutilisé tel quel, sans nouveau composant. Vérification visuelle sur Mac requise.
- [ ] 5.3 Créer `UI/Settings/TrackpadGestureSettings.swift` (carte dans `MouseSettings`, interrupteur avec demande d'Accessibilité, grille des gestes avec badge d'état, picker, avertissements, `MouseExceptionsList(scope: .trackpadGestures)`), ancre `SettingsSectionAnchor.trackpadGestures`, légende et `isOn` dans `MouseSettings`, mots-clés dans `SettingsDirectory` ; décider carte ou page dédiée sur capture ; vérifier sur Mac le rendu dans les langues longues (de, ru, pt-BR).
  - État : carte implémentée ; rendu à vérifier sur Mac dans les langues longues.
- [ ] 5.4 Afficher les actions indisponibles (outil désinstallé, app déplacée, raccourci système désactivé) grisées avec bouton vers Features ; vérifier sur Mac en désinstallant l'outil mappé.
  - État : implémenté ; essai sur Mac requis.
- [x] 5.5 Ajouter la ligne de la feature dans `docs/PERMISSIONS.md` et vérifier le lien et le diff de la doc.

## 6. Vérification d'intégration sur Mac

- [ ] 6.1 Build optimisé `./build.sh`, `./build/Vorssaint --selftest`, `./build.sh --test` complets sans avertissement.
- [ ] 6.2 Scénarios des specs rejoués sur trackpad intégré et Magic Trackpad : chaque geste reconnu, faux positifs rejetés (clic, frappe, repos), exclusion d'app, conflits marqués et libérés, branchement et débranchement, feature désinstallée puis réinstallée avec mappages conservés ; consigner Mac, macOS et ce qui reste non testé dans la PR.
- [ ] 6.3 Calibrer et figer les seuils de balayage et de pincement dans `TrackpadGestureSupport` avec les valeurs mesurées ; relancer la suite `pointer-input`.

## Workflow follow-up

- Ouvrir ou rejoindre vorssaint/vorssaint-utils#2109 pour valider la direction avec le mainteneur avant la PR, et relire vorssaint/vorssaint-utils#2660.
- Ouvrir la PR en brouillon `feat(trackpad): map multi-finger gestures to actions` sans les fichiers `openspec/` et `.claude/commands/opsx`, avec `Refs #2109`.
- Archiver le change OpenSpec une fois la PR fusionnée.
