# Power Platform: praktické cvičení solutions, environments, konfigurace a Git

Plán pro dvě existující prostředí a jedno prázdné řešení v Power Automate. Ověřeno podle dokumentace Microsoftu k 30. 9. 2026. Příkazy jsou pro PowerShell ve Windows.

Cílem je vytvořit flow v DEV, uložit jeho rozbalené soubory do Gitu, sestavit managed balíček a nasadit jej do TEST s jinou konfigurací. Nakonec provedete změnu flow, porovnáte Git diff a nasadíte vyšší verzi.

Odhad času: 3–5 hodin pro základní cvičení; dalších 1–2 hodiny pro SharePoint. Jde o orientační odhad, závislý na připravenosti prostředí a oprávnění.

## 1. Přiřaďte prostředím role a ověřte přístup

Jedno prostředí označte pro účely cvičení jako DEV a druhé jako TEST. Nemusíte měnit jejich skutečné názvy ani typ. TEST je zde role v cvičení; může to být i druhé Developer prostředí.

| Role | Co zde budete dělat | Typ řešení |
| --- | --- | --- |
| DEV | Vytvářet a upravovat flow a definice proměnných | Unmanaged |
| TEST | Importovat balíček, nastavovat cílové hodnoty a testovat | Managed |

V [Power Platform admin center](https://admin.powerplatform.microsoft.com/) otevřete obě prostředí a poznamenejte jejich název, Environment ID a Dataverse Environment URL. Pro CLI použijete ID nebo URL, například `https://organizace-dev.crm4.dynamics.com`; nepoužívejte adresu stránky maker portálu.

Ověřte, že obě prostředí mají plný Dataverse a umožňují práci se Solutions. Samotná existence prostředí ještě neznamená, že má databázi. Pokud v TEST chybí, musíte nejprve zajistit prostředí s Dataverse; exportní postup tento předpoklad nenahrazuje.

Ověřte oprávnění pro vytváření, export a import řešení. Role Environment Maker sama automaticky nezajišťuje přístup k databázi Dataverse. Ve vlastním výukovém prostředí je pro toto cvičení praktická role System Administrator; v podnikových prostředích použijte oprávnění přidělená správcem.

U existujícího Microsoft 365 vývojářského účtu nepředpokládejte automaticky přítomnost všech Power Platform Premium oprávnění. Pro učení je relevantní Power Apps Developer Plan. Samotné základní flow níže nepoužívá žádný premium konektor.

Výstup kroku: znáte DEV a TEST, jejich URL/ID a máte přístup k Solutions v obou.

## 2. Rozlište čtyři pojmy

| Pojem | Význam v tomto cvičení |
| --- | --- |
| Environment | Samostatný kontext pro prostředky, oprávnění, připojení a Dataverse |
| Solution | Přenositelný celek obsahující flow, definice proměnných a později connection reference |
| Environment variable | Pojmenovaná konfigurace: definice je společná, její Current value se může lišit mezi DEV a TEST |
| Connection reference | Součást řešení, kterou v cílovém prostředí namapujete na konkrétní autentizované připojení |

Unmanaged řešení slouží k vývoji. Managed řešení slouží k nasazování. V TEST provádějte změny konfigurace, ale změny logiky flow dělejte v DEV a přenášejte novou verzí. Označení Managed solution je odlišné od funkce Managed Environments.

Gitový commit identifikuje přesný stav souborů. Verze solution je samostatné číslo ve tvaru `major.minor.build.revision`, například `1.0.0.0`. Git tag může tuto verzi propojit s konkrétním commitem; čísla se sama nesynchronizují.

## 3. Připravte stávající prázdné solution

V [Power Automate](https://make.powerautomate.com/) přepněte na DEV a otevřete Solutions → své řešení. Stejné solution lze spravovat i v [Power Apps](https://make.powerapps.com/).

Poznamenejte si jeho skutečný Unique name. Níže používám `EnvironmentVersioning`; pokud je vaše řešení pojmenované jinak, v CLI použijte jeho skutečný Unique name. Název řešení ponechte stejný při všech nasazeních do TEST.

Ve Settings nastavte výchozí verzi `1.0.0.0`. Dokud řešení nic neobsahuje, vytvořte vlastní publisher, například Display name `ALM Lab`, Name `AlmLabPublisher` a Prefix `lab`, a přiřaďte ho řešení. Náhodný prefix výchozího publisheru by také fungoval, ale vlastní prefix usnadní orientaci.

Publisher prefix ovlivňuje schema names nových komponent. V příkladech předpokládám `lab_EnvironmentName` a `lab_MessagePrefix`. Pokud použijete jiný prefix, použijte skutečné názvy z vašeho řešení.

Výstup kroku: prázdné unmanaged řešení v DEV s vlastním publisherem a známým Unique name.

## 4. Vytvořte dvě environment variables

Uvnitř řešení zvolte New → More → Environment variable. Vytvořte:

| Display name | Name / Schema name | Data type | Default value | Current value v DEV | Hodnota pro TEST |
| --- | --- | --- | --- | --- | --- |
| Environment Name | lab_EnvironmentName | Text | Nevyplňovat | DEV | TEST |
| Message Prefix | lab_MessagePrefix | Text | ALM lab | Vývojový pokus | Testovací pokus |

Default value je společná výchozí hodnota zahrnutá v definici. Current value je přepis pro dané prostředí a má přednost. U Environment Name záměrně nepoužívejte DEV jako default: cílové prostředí má dostat výslovnou hodnotu TEST.

Proměnnou prostředí nezaměňujte s akcí Initialize variable uvnitř flow. Ta vytváří proměnnou konkrétního běhu, nikoli konfiguraci spravovanou v řešení.

Výstup kroku: solution obsahuje dvě definice proměnných a DEV má nastavené své hodnoty.

## 5. Vytvořte flow a ověřte hodnoty v DEV

Uvnitř solution zvolte New → Automation → Cloud flow → Instant. Flow pojmenujte `ALM - Configuration Check` a použijte trigger Manually trigger a flow.

Přidejte tři akce Data Operations → Compose:

| Název akce | Inputs |
| --- | --- |
| ReadEnvironment | Environment Name vybraná přes Dynamic content |
| ReadPrefix | Message Prefix vybraná přes Dynamic content |
| Result | Níže uvedený výraz |

Akce přejmenujte před zadáním výrazu. V Result → Expression vložte:

```text
concat('Prostredi=', outputs('ReadEnvironment'), '; zprava=', outputs('ReadPrefix'))
```

Názvy v `outputs(...)` musí odpovídat interním názvům akcí. U proměnných vybírejte položky Dynamic content; nevkládejte text jejich názvů a nevytvářejte ručně identifikátory parametrů.

Uložte a spusťte flow. V Run history rozklikněte Result a ověřte:

```text
Prostredi=DEV; zprava=Vývojový pokus
```

Pokud novou proměnnou nevidíte v Dynamic content, zavřete a znovu otevřete designer. Zkontrolujte také, že flow vytváříte uvnitř správného solution a prostředí.

Výstup kroku: funkční solution-aware flow, které nemá žádné externí připojení.

## 6. Vylučte DEV Current values z exportu

V solution otevřete každou environment variable. U Current value zvolte … → Remove from this solution.

Tato volba vyloučí z exportu hodnotu, ale ponechá ji v DEV. Definice proměnné zůstává součástí řešení. Neodstraňujte celou proměnnou ani její hodnotu z prostředí.

Po tomto kroku znovu spusťte flow v DEV a ověřte stejný výsledek. Přenášet se mají společné definice a logika; TEST dostane vlastní Current values.

Tento krok proveďte před oběma exporty, managed i unmanaged. Při pozdějším přidávání komponent zkontrolujte, zda se hodnoty znovu nestaly součástí solution.

Výstup kroku: DEV stále funguje, exportní balíčky nebudou obsahovat jeho aktuální hodnoty.

## 7. Připravte Git a Power Platform CLI

Na Windows nainstalujte Git, editor podle své volby a .NET SDK. Power Platform CLI lze nainstalovat jako .NET tool:

```powershell
dotnet tool install --global Microsoft.PowerApps.CLI.Tool
pac help
git --version
```

Je-li nástroj už nainstalovaný, aktualizace je `dotnet tool update --global Microsoft.PowerApps.CLI.Tool`. Alternativou je oficiální rozšíření Power Platform Tools ve VS Code. Pokud se `pac` po instalaci nenajde, otevřete nový terminál a ověřte PATH.

V PowerShellu vytvořte pracovní adresář:

```powershell
New-Item -ItemType Directory -Path "$env:USERPROFILE\source\pp-alm-lab" -Force
Set-Location "$env:USERPROFILE\source\pp-alm-lab"
git init -b main
New-Item -ItemType Directory -Path ".\src", ".\config", ".\artifacts" -Force
```

Vytvořte `.gitignore`:

```gitignore
/artifacts/
*.log
*.local.json
```

Vytvořte `README.md` s Unique name řešení, publisher prefixem, rolí DEV/TEST a stručným postupem export → unpack → commit → pack → import. Nesdílejte v něm přihlašovací údaje.

| Cesta | Účel | Verzovat |
| --- | --- | --- |
| src/EnvironmentVersioning/ | Rozbalené soubory řešení | Ano |
| config/dev.settings.json | Netajné hodnoty pro DEV | Ano |
| config/test.settings.json | Netajné hodnoty pro TEST | Ano |
| artifacts/ | Exportované a sestavené ZIPy | Ne; jde o reprodukovatelné balíčky |
| README.md, .gitignore | Postup a pravidla repozitáře | Ano |

Přímá vazba maker portálu na Git není pro tuto cestu potřebná. Power Platform → lokální soubory zajišťuje CLI; verzování a odeslání souborů zajišťuje Git. Takový repozitář může být v GitLabu, GitHubu nebo Azure DevOps.

## 8. Přihlaste CLI, exportujte obě varianty a rozbalte je

Nahraďte ukázkové URL skutečnými Dataverse URL obou prostředí:

```powershell
pac auth create --name DEV --environment "https://organizace-dev.crm4.dynamics.com"
pac auth create --name TEST --environment "https://organizace-test.crm4.dynamics.com"
pac auth list
pac auth select --name DEV
pac auth who
```

Dokončete přihlášení svým Microsoft účtem. Před exportem ověřte, že aktivní profil míří do DEV.

V DEV mají být všechny akce flow uložené. V portálu zvolte Publish all customizations / Publish all changes. Následující exporty proveďte bez mezilehlých změn a pro stejnou verzi:

```powershell
$SolutionName = "EnvironmentVersioning"
pac solution export --name $SolutionName --path ".\artifacts\EnvironmentVersioning.zip" --overwrite
pac solution export --name $SolutionName --path ".\artifacts\EnvironmentVersioning_managed.zip" --managed --overwrite
pac solution unpack --zipfile ".\artifacts\EnvironmentVersioning.zip" --folder ".\src\EnvironmentVersioning" --packagetype Both
```

Unmanaged export je první příkaz; druhý exportuje managed variantu. Názvy ZIPů musí tvořit dvojici `EnvironmentVersioning.zip` a `EnvironmentVersioning_managed.zip` ve stejné složce. Pokud se vaše řešení jmenuje jinak, můžete zachovat tyto lokální názvy souborů; parametr `--name` musí obsahovat skutečný Unique name.

`Both` rozbalí obě varianty do jednoho stromu a uchová jejich rozdíly. Později z něj sestavíte managed balíček. Samotný unmanaged export se tímto nástrojem na managed řešení nepřevádí.

Otevřete `src/EnvironmentVersioning` v editoru. V této export/unpack cestě očekávejte XML metadata, například `Other/Solution.xml`, a definici flow v JSON souboru ve `Workflows`. Přesná struktura závisí na komponentách a verzi nástroje. Nativní Git integrace může používat jiný, YAML formát.

Výstup kroku: čitelné soubory řešení na disku, připravené k verzování.

## 9. Vytvořte konfigurační soubory a první commit

Vygenerujte deployment settings:

```powershell
pac solution create-settings --solution-zip ".\artifacts\EnvironmentVersioning_managed.zip" --settings-file ".\config\test.settings.json"
```

Ponechte skutečné SchemaName vygenerované nástrojem. Pro náš příklad nastavte obsah:

```json
{
  "EnvironmentVariables": [
    { "SchemaName": "lab_EnvironmentName", "Value": "TEST" },
    { "SchemaName": "lab_MessagePrefix", "Value": "Testovací pokus" }
  ],
  "ConnectionReferences": []
}
```

Zkopírujte tento soubor do `config/dev.settings.json` a změňte hodnoty na `DEV` a `Vývojový pokus`. Uložte oba JSONy v UTF-8. DEV soubor zatím slouží k dokumentaci konfigurace; do DEV ho v tomto kroku neimportujete.

Definice environment variables jsou v `src`; požadované aktuální hodnoty jednotlivých prostředí jsou v `config`. Tyto netajné výukové hodnoty můžete verzovat. Hesla, tokeny a privátní klíče do těchto souborů nepatří.

Vytvořte commit a tag:

```powershell
git add src config README.md .gitignore
git diff --cached --stat
git commit -m "Add ALM lab solution 1.0.0.0 and environment configuration"
git tag -a v1.0.0.0 -m "Solution 1.0.0.0"
```

Pokud Git vyžádá identitu autora, nastavte v tomto repozitáři `git config user.name` a `git config user.email` podle svých údajů.

Pro vzdálený Git vytvořte prázdný repozitář ve zvoleném poskytovateli, bez automatického README. Následně:

```powershell
git remote add origin "https://github.com/RadimP/EnvironmentVersioning.git"
git push -u origin main
git push origin v1.0.0.0
```

Výstup kroku: první verzovaný stav řešení a konfigurace.

## 10. Sestavte managed balíček a nasaďte TEST

Z verzovaných souborů sestavte balíček:

```powershell
pac solution pack --folder ".\src\EnvironmentVersioning" --zipfile ".\artifacts\EnvironmentVersioning_from_git_managed.zip" --packagetype Managed
```

Pro první import doporučuji projít průvodce v portálu. V Power Automate přepněte na TEST → Solutions → Import → vyberte sestavený managed ZIP. Zkontrolujte název a verzi a ve výzvě pro proměnné nastavte `TEST` a `Testovací pokus`.

V TEST předem nevytvářejte jiné prázdné řešení stejného jména. Import přinese řešení i jeho komponenty. Pokud cílové prostředí už obsahuje unmanaged variantu tohoto řešení, použijte pro cvičení čistý cíl nebo vyřešte kolizi; managed variantu nelze jednoduše nasadit přes její původní unmanaged řešení.

Po importu zkontrolujte stav flow a případně jej zapněte. Spusťte ho a ověřte:

```text
Prostredi=TEST; zprava=Testovací pokus
```

Pak spusťte flow v DEV a ověřte původní DEV výsledek. V TEST neupravujte Compose akce. Rozdíl chování má pocházet jen z konfigurace.

Alternativní import přes CLI, který použijete u další verze:

```powershell
pac auth select --name TEST
pac auth who
pac solution import --path ".\artifacts\EnvironmentVersioning_from_git_managed.zip" --settings-file ".\config\test.settings.json"
```

Použijte při konkrétním nasazení buď průvodce, nebo CLI. JSON je vstupem CLI importu, nikoli souborem, který nahrajete do běžného průvodce.

Výstup kroku: stejná definice flow pracuje v DEV a TEST s odlišnými hodnotami.

## 11. Proveďte změnu, Git diff a nasazení verze 1.0.1.0

V Gitu založte větev:

```powershell
git switch -c feature/add-version-to-output
```

V DEV změňte Result na:

```text
concat('Prostredi=', outputs('ReadEnvironment'), '; zprava=', outputs('ReadPrefix'), '; verze=1.0.1.0')
```

Uložte flow, spusťte ho a změňte verzi solution na `1.0.1.0` ve Settings. Zkontrolujte vyloučení Current values a publikujte změny.

Přepněte CLI na DEV a opakujte oba exporty z kroku 8. Pro nový unpack použijte prázdnou dočasnou složku, například `artifacts/unpacked-next`, a `--packagetype Both`. Poté nahraďte celý generovaný obsah `src/EnvironmentVersioning` novým stromem. V `src/EnvironmentVersioning` proto neukládejte ručně psané návody ani skripty.

Použití kompletního čerstvého stromu zabrání ponechání souborů komponent, které byly z řešení odstraněny. Před nahrazením musí být původní stav commitnutý a pracovní strom čistý.

Prohlédněte změny:

```powershell
git diff --stat
git diff -- src
git status
git add src
git commit -m "Update configuration check to solution 1.0.1.0"
```

Očekávejte změnu definice flow a metadata verze. Změny GUIDů nebo nečekaně rozsáhlý diff prozkoumejte; často znamenají práci v jiném prostředí nebo opětovné vytvoření komponent.

Sloučení lokální větve:

```powershell
git switch main
git merge --no-ff feature/add-version-to-output
git tag -a v1.0.1.0 -m "Solution 1.0.1.0"
git push origin main
git push origin v1.0.1.0
```

Pokud používáte pull/merge request, proveďte sloučení přes něj a následně aktualizujte lokální main.

Z aktuálního `src` znovu sestavte managed ZIP podle kroku 10. Importujte ho do TEST pomocí CLI a `config/test.settings.json`. Ověřte verzi solution `1.0.1.0` a výsledek:

```text
Prostredi=TEST; zprava=Testovací pokus; verze=1.0.1.0
```

Pro změnu a přidávání komponent stačí Update. Odstranění komponent v cíli řeší Upgrade; běžný Update je neodstraní. Upgrade si nechte jako samostatné rozšíření s cvičnou komponentou. Návrat v Gitu není automatický rollback již nasazeného managed solution.

Výstup kroku: Git obsahuje historii dvou verzí a TEST běží na vyšší verzi se svou konfigurací.

## 12. Změňte pouze konfiguraci v TEST

V TEST nastavte Current value Message Prefix na `Konfigurace změněna v TEST`. Pokud hodnotu nevidíte v managed solution, otevřete její definici v Default solution a upravte pouze Current value.

Po přímé změně hodnoty vypněte a zapněte flow. Microsoft upozorňuje, že flow může jinak používat dřívější hodnotu. Neměňte kvůli tomu logiku managed flow.

Spusťte flow a ověřte novou zprávu při stejné verzi `1.0.1.0`. Poté stejnou hodnotu zapište do `config/test.settings.json` a commitněte ji:

```powershell
git add config/test.settings.json
git commit -m "Change TEST message prefix"
git push origin main
```

Bez aktualizace JSONu by další import s původním deployment settings souborem znovu požadoval starou hodnotu. Git změna konfiguračního souboru sama hodnotu v prostředí nemění; uplatní se až při importu nebo jiném nasazovacím kroku.

Výstup kroku: rozumíte samostatné změně konfigurace, jejímu zachycení v Gitu a obnovení flow.

## 13. Rozšíření: SharePoint a connection references

Vytvořte dvě cvičné SharePoint listy, například `ALM_DEV` a `ALM_TEST`. Mohou být i na stejném webu. Použijte stejnou strukturu se sloupcem Title, ale různé položky, aby šlo snadno poznat, který seznam flow čte.

Poznamenejte URL webu a skutečný GUID každého seznamu, například z List settings URL. Použijte dekódované GUID, nikoli řetězec s URL kódováním.

V DEV solution přidejte dvě další proměnné typu Text:

| Proměnná | DEV Current value | TEST Current value |
| --- | --- | --- |
| lab_SharePointSiteUrl | URL webu DEV seznamu | URL webu TEST seznamu |
| lab_SharePointListId | GUID ALM_DEV | GUID ALM_TEST |

Default values nevyplňujte. Ve flow přidejte SharePoint → Get items. V Site Address a List Name použijte Enter custom value a příslušné environment variables z Dynamic content. Pro první cvičení jen prohlížejte vrácené položky v Run history; není nutné pracovat s dynamickými metadaty vlastních sloupců.

Připojte se k SharePointu. V solution zkontrolujte související connection reference a dejte jí srozumitelný Display name. Pokud se automaticky nevytvořila nebo flow vybralo jinou existující reference, ujistěte se, že používaná reference je zahrnutá v exportovaném solution.

V TEST vytvořte nebo vyberte vlastní dostupné SharePoint connection. Nemusí jít o jiného uživatele; pro cvičení je podstatné přiřazení dostupného cílového connection. Connection reference se přenáší, přihlašovací relace se do ZIPu nebalí.

Znovu vylučte DEV Current values z solution. Zvyšte verzi, exportujte obě varianty, rozbalte, commitněte a sestavte balíček. Vygenerujte nový deployment settings do dočasného souboru a podle něj doplňte stávající konfiguraci, aniž byste přepsali hodnoty předchozích proměnných.

Pro connection reference má cílový JSON tento tvar:

```json
{
  "LogicalName": "SKUTECNY_LOGICKY_NAZEV_REFERENCE_Z_VYGENEROVANEHO_JSONU",
  "ConnectionId": "SKUTECNE_ID_CONNECTION_V_TEST",
  "ConnectorId": "/providers/Microsoft.PowerApps/apis/shared_sharepointonline"
}
```

Tento objekt vložte do pole ConnectionReferences. LogicalName a ConnectorId převezměte z vygenerovaných settings. ConnectionId vezměte z detailu konkrétního connection v TEST; podle dokumentace jej lze zjistit z URL jeho detailu. ID connection z DEV nepoužívejte jako náhradu za cílové connection.

Při ručním importu místo JSONu vyberete cílové connection v průvodci. Při CLI importu musí být connection předem vytvořené, platné a použitelné importujícím uživatelem.

Po nasazení ověřte, že DEV čte ALM_DEV a TEST čte ALM_TEST, přičemž flow má stejnou logiku. Solution nepřenáší samotné SharePoint seznamy ani jejich položky; vytvořili jste je samostatně.

## 14. Volitelné pokračování: nativní Git a CI/CD

CLI + Git cestou výše máte splněné základní zadání. Pokud chcete zkusit také přímé napojení maker portálu, prozkoumejte Source control → Connect to Git.

Nativní Dataverse Git integrace s Azure DevOps vyžaduje Managed Environments, System Administrator v Dataverse a odpovídající Azure DevOps oprávnění/licence. Managed Environments nejsou totéž co managed solution. Nezapínejte tuto funkci jen proto, že jste do TEST importovali managed ZIP.

K 30. 9. 2026 dokumentace uvádí i GitHub integraci v preview; její setup zahrnuje GitHub app a Azure Key Vault. Pro první cvičení představuje další infrastrukturu. Před použitím ověřte aktuální dostupnost a licenční předpoklady. Nativní Git používá jiný souborový formát; nepřipojujte jej bez rozmyslu do stejné složky jako předchozí export/unpack workflow.

Další samostatné cvičení může automatizovat sestavení a import: commit/tag → sestavení managed ZIPu → import s test.settings.json → ověření flow. V produkčnějším postupu oddělte DEV, TEST a PROD, build prostředí a autentizaci nasazování. Pro základní seznámení tato infrastruktura není potřeba.

## 15. Jak prokážete splnění zadání

| Co doložit | Konkrétní výsledek |
| --- | --- |
| Environments | DEV a TEST, v každém samostatná instance flow |
| Solutions | Unmanaged v DEV, managed v TEST |
| Environment variables | Stejná definice, odlišný výstup DEV/TEST |
| Git | Rozbalené XML/JSON soubory, dva commity a tagy verzí |
| Sestavení | Managed ZIP vytvořený příkazem pack z verzovaných souborů |
| Přenos konfigurace | test.settings.json použitý při CLI importu |
| Aktualizace | Vyšší verze solution v TEST s jeho vlastními hodnotami |
| Samostatná konfigurace | Změna hodnoty bez změny logiky flow, zachycená v Gitu |
| Rozšíření SharePoint | Reference namapovaná na connection v TEST; každý flow čte správný seznam |

## 16. Zdroje a řešení častých potíží

| Projev | Co zkontrolovat |
| --- | --- |
| V TEST není Solutions / nelze importovat | Dataverse a databázová oprávnění |
| Proměnná není v Dynamic content | Prostředí, solution-aware flow, nové otevření designeru |
| TEST vypisuje DEV | DEV Current value se přibalila, nebo TEST nemá správné Current value |
| Flow po změně hodnoty vypisuje starou | Vypnout a zapnout flow |
| Pack managed selže | Byly exportovány a rozbaleny obě varianty? Není zdrojový strom pouze unmanaged? |
| Managed import koliduje s řešením | V cíli už existuje originating unmanaged varianta |
| Import prošel, flow je vypnuté | Stav flow, platné connections a oprávnění k nim |
| SharePoint Get items selže | Site URL, GUID listu, přístup connection a shodná struktura |
| Další nasazení vrací staré hodnoty | Aktualizovat cílový deployment settings soubor |
| Po Update zůstala odstraněná komponenta | Rozdíl Update versus Upgrade |

Oficiální dokumentace:

- [Solution concepts](https://learn.microsoft.com/en-us/power-platform/alm/solution-concepts-alm)
- [Power Platform environments](https://learn.microsoft.com/en-us/power-platform/admin/environments-overview)
- [Create a solution and publisher](https://learn.microsoft.com/en-us/power-apps/maker/data-platform/create-solution)
- [Create environment variables](https://learn.microsoft.com/en-us/power-automate/environment-variables)
- [Create a cloud flow in a solution](https://learn.microsoft.com/en-us/power-automate/create-flow-solution)
- [Environment variables in cloud flows](https://learn.microsoft.com/en-us/power-apps/maker/data-platform/environmentvariables-power-automate)
- [Environment variable FAQ — excluding Current value](https://learn.microsoft.com/en-us/power-apps/maker/data-platform/environment-variables-faq)
- [Export a solution](https://learn.microsoft.com/en-us/power-automate/export-flow-solution)
- [Import a solution](https://learn.microsoft.com/en-us/power-automate/import-flow-solution)
- [Install Power Platform CLI](https://learn.microsoft.com/en-us/power-platform/developer/howto/install-cli-net-tool)
- [CLI auth commands](https://learn.microsoft.com/en-us/power-platform/developer/cli/reference/auth)
- [CLI solution commands](https://learn.microsoft.com/en-us/power-platform/developer/cli/reference/solution)
- [SolutionPackager — Both package types](https://learn.microsoft.com/en-us/power-platform/alm/solution-packager-tool)
- [Deployment settings](https://learn.microsoft.com/en-us/power-platform/alm/conn-ref-env-variables-build-tools)
- [Connection references](https://learn.microsoft.com/en-us/power-apps/maker/data-platform/create-connection-reference)
- [Update and upgrade](https://learn.microsoft.com/en-us/power-apps/maker/data-platform/update-solutions)
- [Native Git with Azure DevOps](https://learn.microsoft.com/en-us/power-platform/alm/git-integration/connecting-to-azure-devops)
- [Native Git with GitHub — preview](https://learn.microsoft.com/en-us/power-platform/alm/git-integration/connecting-to-github)
- [Power Platform licensing FAQ](https://learn.microsoft.com/en-us/power-platform/admin/powerapps-flow-licensing-faq)
