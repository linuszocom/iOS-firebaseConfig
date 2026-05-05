# Guide för att hantera GoogleService-Info.plist (och andra känsliga filer) på GitHub

Som många av er har märkt kan GitHub vara lite kinkigt när det gäller filer som innehåller känslig data, som API-nycklar med mera. En sådan fil är `GoogleService-Info.plist`. Denna fil behövs för att vi ska kunna ansluta vår app till Firebase.

I denna guide går vi igenom ett par sätt man kan hantera det på, samt vad man kan göra om skadan redan är skedd.

---

## Metod 1: Lägg GoogleService-filen i `.gitignore`

Detta är förmodligen den enklaste och vanligaste metoden: att helt enkelt lägga `GoogleService-Info.plist` i `.gitignore` *innan* den hamnar på GitHub.

1. Skapa ett nytt iOS-projekt och lägg till Firebase SDK.
2. Skapa en `.gitignore`-fil i projektets rotmapp och lägg till de filer du vill ignorera. Exempel på `.gitignore` för Swift:

```text
# Xcode
DerivedData/
*.xcuserstate
*.xcworkspace
*.xcsettings
!default.xcworkspace
*.xcodeproj/project.xcworkspace/
*.xcodeproj/xcuserdata/
*.xcodeproj/xcshareddata/WorkspaceSettings.xcsettings
*.pbxuser
*.mode1v3
*.mode2v3
*.perspectivev3

# SwiftPM
.build/
Package.resolved

# User-specific
*.swp
*.lock
*.DS_Store
*.idea/
*.moved-aside
*.xcuserdatad/
*.orig

# Playgrounds
timeline.xctimeline
playground.xcworkspace

# CocoaPods
Pods/
Podfile.lock

# Carthage
Carthage/Build/

# Fastlane
fastlane/report.xml
fastlane/Preview.html
fastlane/screenshots
fastlane/test_output

# Firebase / Hemligheter
Secrets.plist
GoogleService-Info.plist

# XCFrameworks
*.xcframework

# SPM artifact cache
.swiftpm/xcode/package-artifacts/

# Asset catalog compilations
*.xcassets

# Environment files
.env
*.env

# Simulator logs and caches
coreSimulator/
```

3. Skapa/hitta ett Firebase-projekt du vill ansluta din iOS-app till.
4. Ladda ner `GoogleService-Info.plist` och lägg filen i rotmappen på ditt projekt.
5. Initiera git i Xcode (eller via terminalen).
6. Lägg till remote och pusha upp till GitHub.

### GoogleService-Info.plist hamnar ändå på GitHub – varför?

Git spårar filer i repot för att kunna upptäcka ändringar. `.gitignore` säger till Git vilka filer som *inte* ska börja spåras.

Men om en fil redan spåras av Git spelar det ingen roll att du senare lägger den i `.gitignore` - den fortsätter vara spårad.

Om du märker att filen spåras (den syns när du kör `git status`) behöver du explicit säga till Git att sluta spåra den innan commit/push:

```bash
git rm --cached GoogleService-Info.plist
```

`--cached` gör att filen tas bort från Gits spårning, men ligger kvar lokalt på din dator.

Sedan:

1. Kontrollera att filen ligger korrekt i `.gitignore`:
   - `GoogleService-Info.plist`
   - eller `MinMapp/GoogleService-Info.plist` om den ligger i undermapp
2. Gör en commit:

```bash
git commit -m "Remove GoogleService-Info.plist from repo"
```

Nu bör du kunna pusha utan att filen följer med.

---

## Metod 2: Konfigurera Firebase utan GoogleService-Info.plist

Ett annat sätt att slippa oroa sig är att inte ha någon plist-fil över huvud taget. Man kan köra konfigurationen programmatiskt istället.

1. Skapa en fil där du vill lagra hemliga variabler, t.ex. `Secrets.plist` (eller en struct/klass).
2. Lägg till den filen i `.gitignore`.
3. Ladda ner `GoogleService-Info.plist`, men lägg **inte** till den i projektet.
4. Öppna plist-filen i en textredigerare, extrahera värdena du behöver och lägg in dem i `Secrets.plist`.

> OBS (SwiftUI): Om appen använder modern `@main App`-struktur utan AppDelegate kan du skapa en `init` i App-structen och kalla `configureFirebase()` därifrån.

Exempel med klassisk AppDelegate:

```swift
class AppDelegate: NSObject, UIApplicationDelegate {
    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil
    ) -> Bool {
        configureFirebase()
        return true
    }

    func configureFirebase() {
        // Gör om Secrets.plist till en dictionary för att komma åt datan
        guard let path = Bundle.main.path(forResource: "Secrets", ofType: "plist"),
              let dict = NSDictionary(contentsOfFile: path) as? [String: Any] else {
            fatalError("Kunde inte läsa Secrets.plist")
        }

        // Skapa FirebaseOptions istället för GoogleService-Info.plist
        let options = FirebaseOptions(
            googleAppID: dict["APP_ID"] as! String,
            gcmSenderID: dict["GCM_SENDER_ID"] as! String
        )
        options.apiKey = dict["API_KEY"] as? String
        options.projectID = dict["PROJECT_ID"] as? String
        options.storageBucket = dict["STORAGE_BUCKET"] as? String

        // Eventuellt fler fält, t.ex.:
        // options.clientID = dict["CLIENT_ID"] as? String

        FirebaseApp.configure(options: options)
    }
}
```

> OBS: Vissa Firebase-funktioner (t.ex. Google Analytics) vill fortfarande ha `GoogleService-Info.plist`, så metoden fungerar inte i alla lägen.

---

## Ta bort känslig data från tidigare commits

Låt säga att du hunnit committa och pusha, och först därefter upptäcker att känslig data har läckt.

Det finns inget direkt sätt att "rensa" historiken på GitHub utan att skriva om historiken. Vanlig lösning är att rensa lokalt och sedan göra en force push. För att rensa filen ur hela Git-historiken rekommenderas `git-filter-repo`.

> ⚠️ Varning: Om ni jobbar flera i samma repo måste alla som redan klonat repot göra en ny ren kloning efteråt. API-nycklar bör också roteras i Firebase om filen hunnit bli publik.

Installera verktyget:

```bash
brew install git-filter-repo
```

Kör i projektroten:

```bash
git filter-repo --path GoogleService-Info.plist --invert-paths
```

Pusha om historiken med force:

```bash
git push origin --force
```

### Command not found: brew?

Det betyder att du inte har **Homebrew** installerat. Det är en pakethanterare för macOS som behövs för att installera `git-filter-repo`.

Så här installerar du Homebrew:

1. Kör detta i terminalen:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

2. Följ instruktionerna på skärmen (den kan be om lösenord och be dig köra kommandon för att lägga till Homebrew i din `PATH`).

När det är klart kan du fortsätta installera `git-filter-repo`.
