# Exploration : gestes multi-doigts sur le trackpad

Mode `openspec explore`, 12 octobre 2026. Ce document capture la reflexion,
pas une proposition. La proposition (`openspec/changes/trackpad-gestures/`)
viendra de `/opsx:propose` une fois le cadrage confirme.

## 1. Le besoin

Mapper des gestes a 3, 4 ou 5 doigts (tap, balayage, pincement) vers des
actions, comme le font BetterTouchTool et Multitouch :

- gestes : tap N doigts, balayage N doigts gauche/droite/haut/bas,
  pincement N doigts ouvrir/fermer, plus tard rotation, tip-tap, coins ;
- actions : raccourci clavier, Spaces gauche/droite, Mission Control,
  App Expose, disposition de fenetre, ouvrir une app ou une URL, touche
  media, outil Vorssaint (Command Bar, presse-papiers, Shelf...), bascule
  rapide, plus tard script shell ;
- reglages : liste d'apps exclues, detection des conflits avec les gestes
  systeme, plus tard des mappages par application.

## 2. Ce que le depot sait deja faire

Decouverte principale : Vorssaint lit deja les contacts bruts du trackpad.

| Brique existante | Ou | Reutilisable pour les gestes |
|---|---|---|
| Pont `dlopen`/`dlsym` vers MultitouchSupport (`MTDeviceCreateList`, `MTRegisterContactFrameCallback`, `MTDeviceStart/Stop`) | `Services/MiddleClick/MiddleClickService.swift` (enum `Multitouch`, prive) | Oui, a extraire dans un service partage |
| Geometrie des touches : centroide + ecartement, offsets 32/36, pas de 96 octets, verifs de plage | idem, `Multitouch.touchGeometry` | Oui, suffit pour tap, balayage, pincement |
| Reconnaisseur de tap 3/4 doigts (duree, deplacement, ecartement, clic physique, frappe clavier recente) | `MiddleClickService.trackTapLocked` + `MiddleClickSupport.tapShouldFire` (pur, teste) | Oui, modele de la machine a etats |
| Hotplug Magic Trackpad (IOKit `AppleMultitouchDevice`), reprise au reveil | `MiddleClickService.installHotplugObserver` | Oui |
| Detection du conflit "glisser a trois doigts" via `CFPreferences` | `MiddleClickService.systemThreeFingerDragEnabled` | Oui, a generaliser aux autres gestes systeme |
| Tap 4 doigts qui ouvre le menu radial | `RadialMenuProfile.trackpadTap`, `MiddleClickSupport.radialMenuTapFingers` | A migrer vers le nouveau moteur |
| Catalogue d'actions Codable : app, file, url, shortcut, tool (15 outils), quickToggle (8), windowLayout (~45), media (4) | `Services/RadialMenu/RadialMenuSupport.swift` (`RadialMenuItem`) | Oui, c'est le modele d'action le plus complet |
| Executeurs de ces actions | `RadialMenuService.run(_:)`, `postShortcut`, `postAuxKey` (prives) | A remonter dans un `ActionRunner` partage |
| Spaces / Mission Control / App Expose via le raccourci symbolique de l'utilisateur | `Services/Switcher/SpaceWindowBridge.spaceShortcut`, `pressSpaceShortcut` | Oui, directement |
| Enregistreur de raccourci et modele `GlobalShortcut` | `UI/ShortcutRecorderButton.swift`, `Core/GlobalShortcut.swift` | Oui |
| Exclusions par application (bundle id) | `Services/MouseExceptions` (`MouseExceptionScope`) | Oui, ajouter un scope `.trackpadGestures` |
| Conventions de tap d'evenements : `SessionActivitySupport.tapShouldRun`, re-armement `tapDisabledByTimeout`, `PointerTapRunLoop` | partout | Oui |

Limites de l'existant :

- un seul callback MultitouchSupport par device : un second abonne (gestes)
  entrerait en conflit avec le middle click. Il faut un distributeur unique ;
- seules les positions x/y sont lues. Identifiants de doigt, angle, pression
  restent non verifies sur Apple Silicon et macOS 14 a 26 ;
- aucun dispatcher central d'actions : chaque panneau s'ouvre par son
  singleton (`CommandBarService.shared.show()`, etc.). Le switcher n'a pas
  d'entree publique ;
- le menu radial et le middle click se partagent deja les taps 3 et 4 doigts,
  avec une regle de priorite ad hoc.

## 3. Contexte amont

- Discussion vorssaint/vorssaint-utils#2109 demande des raccourcis et un
  remappage clavier/souris/trackpad personnalises : c'est la demande la plus
  proche de cette feature.
- La PR vorssaint/vorssaint-utils#2660 (bascule instantanee des Spaces au
  clavier et au trackpad) est ouverte et non relue. Elle synthetise des
  evenements de balayage ; a surveiller pour eviter deux moteurs de gestes.
- CONTRIBUTING demande de discuter la direction dans une issue avant un
  nouveau sous-systeme, et de placer toute nouvelle feature en opt-in
  (`installedByDefault` false).

## 4. Contraintes techniques

Lecture seule. MultitouchSupport observe les contacts, il ne les consomme
pas. Un geste que macOS reconnait aussi (balayage 3 ou 4 doigts, pincement
4 doigts, tap 3 doigts pour Rechercher) declenchera les deux actions. BTT et
Multitouch demandent a l'utilisateur de desactiver le geste systeme. La bonne
strategie est donc :

1. proposer en priorite les gestes libres : 5 doigts (tap, balayage,
   pincement), tap 4 doigts, tap 3 doigts quand Rechercher est desactive ;
2. lire les preferences systeme (`com.apple.AppleMultitouchTrackpad`,
   `com.apple.driver.AppleBluetoothMultitouch.trackpad`, `com.apple.dock`)
   pour marquer chaque geste "pris par macOS" et afficher le chemin de
   Reglages Systeme a suivre, comme le fait deja le middle click. Cles a
   verifier sur Mac : `TrackpadThreeFingerHorizSwipeGesture`,
   `TrackpadFourFingerHorizSwipeGesture`, `TrackpadThreeFingerVertSwipeGesture`,
   `TrackpadFourFingerVertSwipeGesture`, `TrackpadFourFingerPinchGesture`,
   `TrackpadFiveFingerPinchGesture`, `TrackpadThreeFingerTapGesture`,
   `TrackpadThreeFingerDrag`, et cote Dock `showMissionControlGestureEnabled`,
   `showAppExposeGestureEnabled`, `showDesktopGestureEnabled`,
   `showLaunchpadGestureEnabled` ;
3. ne jamais tenter de supprimer un geste systeme par un tap CGEvent : le
   WindowServer le gere avant nous et le resultat n'est pas fiable.

Autres contraintes :

- permission : Accessibilite seulement (deja le cas du middle click), pas
  d'Input Monitoring dans le projet ;
- distribution : hors App Store, notarisation Developer ID. Aucun refus connu
  pour un chargement `dlopen` d'un framework prive ; la sandbox est exclue ;
- struct de touche : rester sur x/y (centroide, ecartement, angle calcule a
  partir des positions) pour ne pas dependre d'offsets non verifies ;
- localisation : 15 langues obligatoires a la compilation, tests de qualite ;
- tests : binaire de test maison, seuls les fichiers `*Support.swift` et
  `*Strings.swift` sont compiles. La reconnaissance doit etre pure ;
- verification : cet environnement est Linux, sans Swift ni trackpad. Rien
  ne peut etre compile ni essaye ici. Un Mac reel (macOS 14, 15, 26, trackpad
  integre et Magic Trackpad) est indispensable avant toute PR.

## 5. Architecture envisagee

```
  MultitouchSupport (prive, dlopen)
          |  frames (x, y par doigt, N doigts, horodatage)
          v
+-------------------------------+
| TrackpadTouchService          |  nouveau : extrait de MiddleClickService
|  - liste des devices, hotplug |  un seul callback par device
|  - lecture x/y, geometrie     |  distribue aux abonnes (thread multitouch)
+-------------------------------+
     |              |                 |
     v              v                 v
 MiddleClick   RadialMenu tap    TrackpadGestureService (nouveau)
 (clic 3/4)    (migre ici)            |
                                      v
                         +---------------------------+
                         | TrackpadGestureSupport    |  pur, teste
                         |  machine a etats :        |
                         |  idle -> candidat(N)      |
                         |   -> tap | swipe(dir)     |
                         |   -> pinch(in/out)        |
                         |   -> annule (clic, +doigt)|
                         +---------------------------+
                                      |  GestureEvent{kind, fingers, dir}
                                      v
                         +---------------------------+
                         | mappage [gesture: action] |  JSON Data dans UserDefaults
                         | + exclusions par app      |  scope MouseExceptions
                         +---------------------------+
                                      |  main thread
                                      v
                         +---------------------------+
                         | ActionRunner (partage)    |  extrait de RadialMenuService
                         |  shortcut, spaces, layout,|  + SpaceWindowBridge
                         |  tool, media, app, url    |
                         +---------------------------+
```

Reconnaissance a partir du seul centroide et de l'ecartement :

- tap : N doigts posent et levent en moins de 0,35 s, deplacement < 0,03,
  ecartement stable (seuils deja calibres dans `MiddleClickSupport`) ;
- balayage : deplacement du centroide > seuil dans un axe dominant, juge a
  la levee des doigts (une seule action par contact) ;
- pincement : variation d'ecartement > seuil, centroide quasi fixe ;
- annulation : clic physique, nombre de doigts depasse, frappe clavier
  recente, positions illisibles (fail closed, comme aujourd'hui).

## 6. Options comparees

| Option | Contenu | Pour | Contre |
|---|---|---|---|
| A. Minimal | Ajouter des taps 5 doigts au service middle click | Peu de code | Pas de balayage, dette dans un fichier deja charge |
| B. Moteur partage (recommandee) | `TrackpadTouchService` + reconnaisseur pur + mappage vers `RadialMenuItem` + `ActionRunner` partage, exclusions par app, conflits systeme affiches | Reutilise 80 % de l'existant, testable, unifie middle click et menu radial | Refactor transversal de deux services, verification sur Mac obligatoire |
| C. BTT complet | B + mappages par app, modificateurs, gestes continus, HUD, scripts | Parite avec BTT | Trop large pour une PR, decisions produit nombreuses |

Recommandation : B pour une premiere version, avec C en jalons separes
(par app, puis modificateurs et rotation).

## 7. Perimetre propose pour la v1

Gestes : tap 3/4/5 doigts, balayage 3/4/5 doigts dans les quatre directions,
pincement 4/5 doigts ouvrir/fermer. Chaque geste porte un etat "libre" ou
"pris par macOS" avec le chemin pour le liberer.

Actions : raccourci clavier, Space gauche/droite, Mission Control, App Expose,
disposition de fenetre, outil Vorssaint (y compris Command Bar et switcher,
qui demandent une entree publique), touche media, bascule rapide, ouvrir une
app ou une URL.

Reglages : nouvelle feature opt-in `trackpadGestures` dans le groupe souris et
clavier, carte dans la page Souris (ou page dediee si la grille est trop
dense), liste d'apps exclues, sauvegarde des reglages automatique via les cles
enregistrees.

Hors v1 : mappages par application, modificateurs requis, rotation, tip-tap,
coins, scripts shell, retour haptique, repetition continue.

## 8. Questions ouvertes

1. Faut-il ouvrir ou rejoindre une issue amont (la discussion #2109) avant
   d'implementer, comme CONTRIBUTING le demande ?
2. Le tap middle click et le tap du menu radial migrent-ils dans le nouveau
   moteur (une seule source de verite pour les taps) ou restent-ils tels quels
   avec une regle de priorite ?
3. Carte dans la page Souris ou page Reglages dediee ?
4. Qui verifie sur Mac reel, et sur quelles versions de macOS ?
5. Le mappage par application est-il attendu des la premiere version ?
