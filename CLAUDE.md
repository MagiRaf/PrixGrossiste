# CLAUDE.md - PrixGrossisteCS

Mémoire du projet pour Claude Code. Ce fichier documente les décisions, problèmes rencontrés et l'état actuel.

## Commandes

```bash
# Build Debug (développement)
dotnet build --configuration Debug

# Lancer (Debug)
./PrixGrossiste/bin/Debug/net8.0-windows/win-x64/PrixGrossiste.exe

# Build SingleFile (déploiement) - PublishSingleFile est dans le .csproj en Release
dotnet publish -c Release -o PrixGrossiste/bin/SingleFile

# Lancer (SingleFile)
./PrixGrossiste/bin/SingleFile/PrixGrossiste.exe
```

## Version actuelle
- **Version** : `0.0044` (définie dans `App.xaml.cs` constante `AppVersion`)
- **Format** : nombre incrémental (pas semver), ex: 0.0043 → 0.0044
- **Nom exe** : `PrixGrossiste.exe` (Debug ET SingleFile, standardisé le 08/02/2026)

## Architecture de déploiement

### Build SingleFile
Le dossier `bin/SingleFile/` contient le minimum pour déployer :
- `PrixGrossiste.exe` (~172 Mo, self-contained .NET 8 + toutes les DLL, WebView2Loader embarqué)
- `credentials.json` (identifiants par compte/fournisseur)
- `settings.json` (créé au premier lancement)

**Fichiers NON nécessaires pour le déploiement** : `.pdb`, `.xml`, `version.txt`

### Contenu du ZIP pour Google Drive
Le ZIP à uploader sur Google Drive doit contenir un dossier `PrixGrossiste/` avec :
```
PrixGrossiste/
├── PrixGrossiste.exe
└── credentials.json
```

### Dépendance WebView2 Runtime
L'app a besoin du WebView2 Runtime (basé sur Edge Chromium). Il est :
- Pré-installé sur Windows 10 21H2+ et Windows 11
- Sinon installable via `MicrosoftEdgeWebview2Setup.exe` (inclus dans le zip)
- **Vérifié automatiquement au démarrage** dans `App.xaml.cs` via le registre Windows
- Si absent, propose l'installation silencieuse (`/silent /install`)

### Vérification fichiers requis au démarrage (App.xaml.cs)
Au démarrage, l'app vérifie que `credentials.json` existe.
Si manquant, propose un téléchargement automatique depuis Google Drive.

## Système de mise à jour

### Deux mécanismes coexistent

#### 1. UpdateService.cs (mise à jour de version)
- Vérifie `version.ini` sur GitHub raw au démarrage
- Si nouvelle version, propose le téléchargement du ZIP depuis Google Drive
- Télécharge, extrait, génère un `update.bat` puis ferme l'app
- Le batch sauvegarde credentials.json + settings.json, remplace les fichiers, restaure, relance

#### 2. App.xaml.cs (réparation fichiers manquants)
- Au tout premier démarrage, vérifie les fichiers requis
- Si manquants, télécharge le ZIP complet depuis Google Drive (URL différente)
- Copie les fichiers sans toucher à l'exe en cours

### URLs configurées
- **Version** : `https://raw.githubusercontent.com/MagiRaf/PrixGrossiste/main/version.ini`
  - Format : `[Version]\nversion=0.0044`
- **ZIP mise à jour (UpdateService)** : `https://drive.google.com/uc?export=download&confirm=1&id=1LfZPbZv4pW0GaJfATjIMVt2s_O005a_w`
- **ZIP réparation (App.xaml.cs)** : `https://drive.google.com/uc?export=download&confirm=t&id=1G8pf9fq5zsmAiMt5UCHP2vEi_R6iAPzj`

### Workflow pour déployer une mise à jour
1. Incrémenter `AppVersion` dans `App.xaml.cs`
2. `dotnet publish -c Release -o PrixGrossiste/bin/SingleFile`
3. `powershell -ExecutionPolicy Bypass -File make_zip.ps1` (crée PrixGrossiste.zip)
4. Uploader le ZIP sur Google Drive (même ID de fichier = remplacer)
5. Mettre à jour `version.ini` sur GitHub : `gh api repos/MagiRaf/PrixGrossiste/contents/version.ini -X PUT -f message="version X.XXXX" -f content="<base64>" -f sha="<sha>"`
6. Mettre à jour `CLAUDE.md` sur GitHub : même commande avec le contenu du fichier local
7. Les utilisateurs verront la mise à jour au prochain lancement

### Bugs corrigés (session 08/02/2026)
1. ~~credentials.json et settings.json supprimés par le batch~~ → sauvegarde/restauration ajoutée
2. ~~Nom exe incohérent~~ → standardisé sur `PrixGrossiste.exe`
3. ~~PublishSingleFile pas dans le .csproj~~ → ajouté en condition Release
4. ~~RequiredFiles référençait PrixGrossisteApp.dll~~ → corrigé pour SingleFile

### Note Google Drive gros fichiers
Pour les fichiers >100Mo, Google peut afficher une page "Virus scan warning".
Le paramètre `&confirm=1` (ou `&confirm=t`) dans l'URL contourne cette page.

## Gestion des comptes (session 08/02/2026)

### Cookies et sessions
- Chaque WebView a son propre dossier de données dans `%TEMP%\PrixGrossiste\{compte}\{fournisseur}\`
- **Problème découvert** : quand on change de compte à l'exécution, les WebViews gardent l'ancien dossier. Les cookies du nouveau compte sont stockés dans le dossier de l'ancien.
- **Solution appliquée** : `ClearCookiesAsync()` supprime les cookies avant chaque login (au démarrage ET au changement de compte)
- `LogoutAsync()` est appelé avant `NavigateToLoginAsync()` lors d'un changement de compte
- Les cookies sont négligeables en taille (quelques Ko par site)

### Searchers avec login custom
Ces searchers ont leur propre méthode de login (pas `AutoLoginAsync` de base) :
- **KMLSSearcher** : `AutoLoginKMLSAsync()` - utilise toujours Met4vape (Lynkeco n'a pas de compte KMLS)
- **JoshNoaSearcher** : `AutoLoginJoshNoaAsync()` - redirige vers "/" au lieu de /mon-compte
- **LCASearcher** : `AutoLoginLCAAsync()` - gère les cookies et 2FA

Tous les trois appellent `ClearCookiesAsync()` au début de leur login.

## Score de correspondance (session 08/02/2026)

### Problème "Score trop faible" sans prix
Quand le score < 70%, les searchers retournaient "N/A" pour le prix même si le produit était trouvé.
Exemple : "qf meshed" → produit "Résistances QF par 3 - Vaporesso" trouvé mais prix masqué car score 31%.

### Solution appliquée
- **Groupe A** (prix dispo dans la liste : GFC, ADNS, GreenVillage, JoshNoa) : remplacé `"N/A"` par `CleanPrice(bestPrice)`
- **Groupe B** (prix via clic sur produit : CigAccess, LVP, Eclopediscount, KMLS, LCA) : déplacé le check score APRÈS la récupération du prix
- **Lemotion** (hybride) : même traitement
- Note : pour le Groupe B, la recherche est un peu plus lente car l'app clique sur le produit même avec un score faible

### Bonus correspondance littérale (session 10/02/2026)
Quand `pod` est normalisé en `cartouche`, un produit "Fusion Pod System" matchait autant que "Cartouche Fusion Pod System". Le produit avec "Cartouche" littéral dans le nom était même pénalisé (plus de mots → pénalité mots en plus).

**Solution** : bonus +5 points par mot de la recherche trouvé **tel quel** (avant normalisation) dans le nom original du produit. Ainsi "Cartouche Fusion Pod System" reçoit +5 pour "cartouche" littéral, ce qui le favorise.

### Seuil de score
- Score >= 70% : produit affiché normalement avec prix
- Score < 70% : préfixé "Score trop faible:" avec le vrai prix
- `ShowLowScores` par défaut = `true` (changé le 08/02/2026)

## Settings par défaut (SettingsService.cs)
```
Language = "FR"
Theme = "Dark"
LastAccount = "Lynkeco"
ShowLowScores = true
ZoomLevel = 1.1
```

## URLs des fournisseurs

| Fournisseur | URL |
|-------------|-----|
| GFC | https://www.gfc-provap.com/fr/ |
| ADNS | https://adns-grossiste.fr/ |
| CigAccess | https://www.cigaccess-pro.com/ |
| Lemotion | https://www.grossisteecigarette.fr/ |
| Eclopediscount | https://www.eclopediscount.com/ |
| JoshNoa | https://joshnoaco.fr/ |
| LVP | https://www.lvp-distribution.fr/ |
| GreenVillage | https://greenvillage-grossiste.fr/ |
| LCA | https://www.lca-distribution.com/ |
| KMLS | https://www.kmls.fr/fr/ |

## Notes techniques

### Dispatcher et async (important)
```csharp
// Pour les opérations async avec Dispatcher.InvokeAsync :
await Application.Current.Dispatcher.InvokeAsync(async () => {
    return await SomeAsyncOperation();
}).Task.Unwrap();  // <-- .Task.Unwrap() est crucial !
```

### Login avec nativeInputValueSetter
Pour contourner React/Vue qui interceptent les modifications de valeur :
```javascript
var setter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
setter.call(element, 'valeur');
element.dispatchEvent(new Event('input', {bubbles: true}));
```

### Détection de connexion
Combinaison de 2 méthodes :
1. Présence d'un lien logout/compte
2. URL qui ne contient plus /login, /connexion, /authentification

### WebView2 et ExecuteScript
- Les WebViews doivent être dans une fenêtre visible pour fonctionner
- Mode invisible = `Visibility.Collapsed` (pas visible mais actif)
- `ClearBrowsingDataAsync(Cookies)` pour supprimer les cookies d'un profil

### Dossiers WebView2
- Chemin : `%TEMP%\PrixGrossiste\{compte}\{fournisseur}\`
- Chaque fournisseur a son propre dossier isolé
- Peut être supprimé pour reset complet

## Images produit (session 10/02/2026)

### Fonctionnalité
- Images miniatures (40x40px) affichées inline dans la colonne produit
- Récompense de niveau 5 dans le système de gamification
- `ImageUrl` ajouté au record `SearchResult` et à la classe `SupplierResult`

### Deux stratégies d'extraction

**URL directe** (searchers qui récupèrent le prix depuis la liste) :
- GFC : `.product_img_container img, .product_img_link img`
- ADNS : `.se_img img` avec priorité `data-src` (lazy loading), filtre SVG, URLs relatives → absolues
- JoshNoa, GreenVillage : `img` standard dans la carte produit
- KMLS : `img` dans `.ais-Hits-item`

**Base64 canvas** (searchers qui naviguent vers la page produit - contourne le 403 hotlink) :
- CigAccess, Eclopediscount, Lemotion, LVP, LCA
- Technique : `canvas.toDataURL('image/jpeg', 0.7)` dans le WebView après chargement de la page
- Le WebView a les cookies/headers nécessaires, WPF's BitmapImage non

### ImageUrlConverter (MainWindow.xaml.cs)
Converter WPF qui gère les deux formats :
- `data:image/...` → décode base64 en `BitmapImage` via `MemoryStream`
- `https://...` → `BitmapImage` classique avec `UriSource`
- `DecodePixelWidth = 60` pour optimiser la mémoire

### Problèmes rencontrés
- **GFC** : `querySelector('img')` capturait le logo marque → sélecteur spécifique `.product_img_container img`
- **ADNS lazy loading** : `src` contient un SVG placeholder, vraie URL dans `data-src`
- **ADNS URLs relatives** : `/57535-home_default/...` → préfixer `https://adns-grossiste.fr`
- **403 Forbidden** (CigAccess, Eclopediscount, Lemotion partiel) : hotlink protection bloque les requêtes HTTP sans cookies/Referer → solution base64 canvas

## Système de gamification - Niveaux (session 10/02/2026)

### Décalage des niveaux
Les récompenses ont été décalées pour insérer "Images produit" au niveau 5 :

| Niv | Récompense | Avant | Après |
|-----|------------|-------|-------|
| 5 | Images produit | (n'existait pas) | **NOUVEAU** |
| 6 | Historique des prix | était niv 5 | niv 6 |
| 7 | Choix de classe | était niv 6 | niv 7 |
| 8 | Thèmes personnalisés | était niv 7 | niv 8 |
| 11 | Easter egg téléphone | était niv 10 | niv 11 |

### Titre Requin
`Title_Diamant` remplacé par `Title_Requin` (🦈) dans GameService et TranslationService.
