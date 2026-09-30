# EnvironmentVersioning — Power Platform ALM

Výukový repozitář pro správu Power Platform solution, verzování jeho rozbalených souborů v Gitu a nasazování mezi dvěma prostředími s odlišnou konfigurací.

## Identifikace řešení

| Vlastnost | Hodnota |
| --- | --- |
| Solution Unique name | `EnvironmentVersioning` |
| Publisher prefix | `lab` |
| Výchozí verze pro cvičení | `1.0.0.0` |
| DEV Environment ID / Dataverse URL | **DOPLNIT** |
| TEST Environment ID / Dataverse URL | **DOPLNIT** |

**Před použitím ověřte Unique name a publisher prefix podle skutečného solution.** Hodnoty `EnvironmentVersioning` a `lab` pocházejí z příkladu cvičení. Pokud se liší, upravte názvy a schema names v konfiguraci. Lokální názvy ZIPů a adresářů v příkazech můžete ponechat.

## Prostředí

| Role | Účel | Typ solution |
| --- | --- | --- |
| DEV | Vytváření a úpravy flow a definic environment variables | Unmanaged |
| TEST | Import managed balíčku, cílová konfigurace a ověření chování | Managed |

Změny logiky provádějte v DEV a do TEST je přenášejte novou verzí solution. V TEST nastavujte jeho vlastní Current values a přiřazení connection references.

## Struktura repozitáře

| Cesta | Účel | Verzovat |
| --- | --- | --- |
| `src/EnvironmentVersioning/` | Rozbalené soubory řešení; pro tento postup varianty Managed a Unmanaged rozbalené pomocí `Both` | Ano |
| `config/dev.settings.json` | Netajné hodnoty pro DEV | Ano |
| `config/test.settings.json` | Netajné hodnoty pro TEST | Ano |
| `artifacts/` | Exportované a sestavené ZIPy, dočasné výstupy | Ne; jde o reprodukovatelné balíčky |
| `README.md`, `.gitignore` | Postup a pravidla repozitáře | Ano |

Do `.gitignore` vložte:

```gitignore
/artifacts/
*.log
*.local.json
```

Do `src/EnvironmentVersioning/` ukládejte pouze generované soubory solution. Ručně psané návody a skripty patří mimo tento adresář.

## Předpoklady

- Git a Power Platform CLI dostupné příkazy `git --version` a `pac help`.
- Dataverse a oprávnění pro práci se solutions v obou prostředích.
- Uložené a otestované flow v DEV solution.
- Vyplněný `config/test.settings.json` pro cílové prostředí.

Příkazy spouštějte v PowerShellu z kořene tohoto repozitáře.

Nastavte skutečné údaje a přihlaste CLI:

```powershell
$SolutionName = "EnvironmentVersioning"
$DevEnvironment = "DOPLNIT_ENVIRONMENT_ID_NEBO_DATAVERSE_URL_DEV"
$TestEnvironment = "DOPLNIT_ENVIRONMENT_ID_NEBO_DATAVERSE_URL_TEST"

pac auth create --name DEV --environment $DevEnvironment
pac auth create --name TEST --environment $TestEnvironment
```

Přihlášení proběhne interaktivně. Hesla a tokeny do příkazů ani do repozitáře nevkládejte.

## Postup: export → unpack → commit → pack → import

### 1. Export z DEV

V DEV uložte změny a publikujte je přes **Publish all changes / Publish all customizations**. Nastavte verzi solution, například `1.0.0.0`.

U environment variables zvolte u **Current value → … → Remove from this solution**. DEV hodnota zůstane v prostředí, ale nebude součástí exportu. Definice proměnné ponechte v solution.

```powershell
New-Item -ItemType Directory -Path ".\src", ".\config", ".\artifacts" -Force

pac auth select --name DEV
pac auth who

pac solution export --name $SolutionName --path ".\artifacts\EnvironmentVersioning.zip" --overwrite
pac solution export --name $SolutionName --path ".\artifacts\EnvironmentVersioning_managed.zip" --managed --overwrite
```

Ověřte úspěch obou exportů. Exportujte obě varianty stejné verze bez mezilehlých změn. ZIPy musí být ve stejné složce a mít párové názvy `EnvironmentVersioning.zip` a `EnvironmentVersioning_managed.zip`.

### 2. Unpack

Při prvním rozbalení použijte prázdný cílový adresář:

```powershell
pac solution unpack --zipfile ".\artifacts\EnvironmentVersioning.zip" --folder ".\src\EnvironmentVersioning" --packagetype Both
```

`Both` uchová rozdíly Managed a Unmanaged variant pro následné sestavení. Pouhý unmanaged export se tímto nástrojem na managed řešení nepřevádí.

Při dalších exportech rozbalte novou verzi do nové prázdné složky pod `artifacts/` a jejím obsahem nahraďte celý generovaný strom `src/EnvironmentVersioning/`. Před nahrazením musí být původní stav commitnutý. Tím se vyhnete ponechání souborů odstraněných komponent.

### 3. Konfigurace a commit

Při prvním nasazení vygenerujte šablonu deployment settings:

```powershell
pac solution create-settings --solution-zip ".\artifacts\EnvironmentVersioning_managed.zip" --settings-file ".\config\test.settings.json"
```

Doplňte cílové hodnoty. Pro základní cvičení:

| Schema name | DEV | TEST |
| --- | --- | --- |
| `lab_EnvironmentName` | `DEV` | `TEST` |
| `lab_MessagePrefix` | `Vývojový pokus` | `Testovací pokus` |

Ponechte skutečné schema names vygenerované z vašeho solution. Vytvořte také `config/dev.settings.json` s hodnotami DEV. Pokud flow používá konektory, doplňte do TEST konfigurace ID dostupných connections v TEST.

Při změně komponent generujte novou šablonu do dočasného souboru pod `artifacts/` a změny slučte do existující konfigurace; nepřepisujte již vyplněné cílové hodnoty.

```powershell
git diff -- src config
git add src config README.md .gitignore
git diff --cached --stat
git commit -m "Add solution 1.0.0.0 and environment configuration"
git tag -a v1.0.0.0 -m "Solution 1.0.0.0"
```

Číslo v commit zprávě a tagu přizpůsobte skutečné verzi solution. Je-li připojený remote `origin`, odešlete commit a tag:

```powershell
git push -u origin HEAD
git push origin v1.0.0.0
```

### 4. Pack managed balíčku

Sestavte balíček z commitnutých zdrojů:

```powershell
pac solution pack --folder ".\src\EnvironmentVersioning" --zipfile ".\artifacts\EnvironmentVersioning_from_git_managed.zip" --packagetype Managed
```

### 5. Import do TEST

Po úspěšném sestavení ověřte cílový profil a importujte s cílovou konfigurací:

```powershell
pac auth select --name TEST
pac auth who

pac solution import --path ".\artifacts\EnvironmentVersioning_from_git_managed.zip" --settings-file ".\config\test.settings.json"
```

V TEST zkontrolujte verzi solution, stav flow a výsledek zkušebního běhu. U základního flow očekávejte:

```text
Prostredi=TEST; zprava=Testovací pokus
```

Ověřte také, že DEV stále používá své hodnoty.

## Další změny

- Změňte flow v DEV, otestujte ho a zvyšte verzi solution ve formátu `major.minor.build.revision`.
- Zopakujte export obou variant, čerstvý unpack, kontrolu Git diff, commit, pack a import.
- Při změně konfigurace TEST aktualizujte také `config/test.settings.json`, aby další nasazení použilo požadované hodnoty.
- Po přímé změně environment variable v TEST flow vypněte a zapněte, aby načetlo novou hodnotu.
- Odstranění komponent v cíli řeší Upgrade; běžný Update je v cílovém prostředí neodstraní.

## Přihlašovací údaje

README, konfigurační soubory i Git smějí obsahovat pouze netajné údaje. Neukládejte hesla, access tokeny, client secrets ani privátní klíče. Connection reference se mapuje na existující connection; jeho přihlašovací údaje nejsou součástí verzované konfigurace.

## Dokumentace

- [Power Platform CLI — solution](https://learn.microsoft.com/en-us/power-platform/developer/cli/reference/solution)
- [Power Platform CLI — auth](https://learn.microsoft.com/en-us/power-platform/developer/cli/reference/auth)
- [SolutionPackager — Managed a Unmanaged varianty](https://learn.microsoft.com/en-us/power-platform/alm/solution-packager-tool)
- [Deployment settings](https://learn.microsoft.com/en-us/power-platform/alm/conn-ref-env-variables-build-tools)
- [Environment variables — vyloučení Current value z exportu](https://learn.microsoft.com/en-us/power-apps/maker/data-platform/environment-variables-faq)
