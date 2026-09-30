# Power Platform: praktické cvičení solutions, environments, konfigurace a Git

Plán pro dvě existující prostředí a jedno prázdné řešení v Power Automate. Ověřeno podle dokumentace Microsoftu k 30. 9. 2026. Příkazy jsou pro PowerShell ve Windows.

Cílem je vytvořit flow v DEV, uložit jeho rozbalené soubory do Gitu, sestavit managed balíček a nasadit jej do TEST s jinou konfigurací. Nakonec provedete změnu flow, porovnáte Git diff a nasadíte vyšší verzi. Rozšířený bod 14 převádí tento lab na nativní Dataverse Git integraci s Azure Repos a CI/CD v Azure DevOps Pipelines.

Odhad času: 3–5 hodin pro základní cvičení; dalších 1–2 hodiny pro SharePoint. Nativní Git a Azure DevOps v bodu 14 si rozdělte do dalších několika pracovních bloků; čekání na licence či přidělení agenta se do nich nezapočítává. Jde o orientační odhad, závislý na připravenosti prostředí a oprávnění.

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

Poznamenejte si jeho skutečný Unique name. Níže používám `PPAlmLab`; pokud je vaše řešení pojmenované jinak, v CLI použijte jeho skutečný Unique name. Název řešení ponechte stejný při všech nasazeních do TEST.

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
| src/PPAlmLab/ | Rozbalené soubory řešení | Ano |
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
$SolutionName = "PPAlmLab"
pac solution export --name $SolutionName --path ".\artifacts\PPAlmLab.zip" --overwrite
pac solution export --name $SolutionName --path ".\artifacts\PPAlmLab_managed.zip" --managed --overwrite
pac solution unpack --zipfile ".\artifacts\PPAlmLab.zip" --folder ".\src\PPAlmLab" --packagetype Both
```

Unmanaged export je první příkaz; druhý exportuje managed variantu. Názvy ZIPů musí tvořit dvojici `PPAlmLab.zip` a `PPAlmLab_managed.zip` ve stejné složce. Pokud se vaše řešení jmenuje jinak, můžete zachovat tyto lokální názvy souborů; parametr `--name` musí obsahovat skutečný Unique name.

`Both` rozbalí obě varianty do jednoho stromu a uchová jejich rozdíly. Později z něj sestavíte managed balíček. Samotný unmanaged export se tímto nástrojem na managed řešení nepřevádí.

Otevřete `src/PPAlmLab` v editoru. V této export/unpack cestě očekávejte XML metadata, například `Other/Solution.xml`, a definici flow v JSON souboru ve `Workflows`. Přesná struktura závisí na komponentách a verzi nástroje. Nativní Git integrace může používat jiný, YAML formát.

Výstup kroku: čitelné soubory řešení na disku, připravené k verzování.

## 9. Vytvořte konfigurační soubory a první commit

Vygenerujte deployment settings:

```powershell
pac solution create-settings --solution-zip ".\artifacts\PPAlmLab_managed.zip" --settings-file ".\config\test.settings.json"
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
git remote add origin "SEM_VLOZTE_URL_SVEHO_REPOZITARE"
git push -u origin main
git push origin v1.0.0.0
```

Výstup kroku: první verzovaný stav řešení a konfigurace.

## 10. Sestavte managed balíček a nasaďte TEST

Z verzovaných souborů sestavte balíček:

```powershell
pac solution pack --folder ".\src\PPAlmLab" --zipfile ".\artifacts\PPAlmLab_from_git_managed.zip" --packagetype Managed
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
pac solution import --path ".\artifacts\PPAlmLab_from_git_managed.zip" --settings-file ".\config\test.settings.json"
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

Přepněte CLI na DEV a opakujte oba exporty z kroku 8. Pro nový unpack použijte prázdnou dočasnou složku, například `artifacts/unpacked-next`, a `--packagetype Both`. Poté nahraďte celý generovaný obsah `src/PPAlmLab` novým stromem. V `src/PPAlmLab` proto neukládejte ručně psané návody ani skripty.

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

## 14. Nativní Dataverse Git integrace a CI/CD v Azure DevOps

Tento postup navazuje na `PPAlmLab` a dvě prostředí DEV/TEST. Výsledkem bude vývoj v maker portálu, commity přímo z Dataverse do Azure Repos, kontrola změn v pull requestu a sestavení a nasazení přes Azure DevOps Pipelines. Stejné unmanaged řešení z DEV můžete použít i zde; „nativní řešení“ není nový typ solution, ale způsob propojení jeho zdrojů s Gitem.

Příklady předpokládají Unique name `PPAlmLab`, publisher `AlmLabPublisher` a prefix `lab`. Nahraďte je skutečnými názvy. Pro nový postup doporučuji samostatný repozitář `pp-alm-native`. Předchozí GitHub repozitář si ponechte jako záznam prvního cvičení. Nativní formát zdrojů se liší od rozbaleného XML z bodů 8–10, proto oba postupy neposílejte do stejné zdrojové složky.

### 14.1 Jak budou části spolupracovat

| Část | Úloha |
| --- | --- |
| Dataverse DEV | Autorování flow a proměnných v unmanaged řešení |
| Dataverse Git integrace | Commit změn z DEV do Azure Repos; pull změn zpět do DEV |
| Azure Repos | Historie zdrojů, větve, pull requesty a verze pipeline |
| Azure Pipelines – Build | Sestavení managed ZIPu ze souborů konkrétního commitu |
| Azure Pipelines – Deploy_TEST | Import stejného ZIPu a jeho TEST konfigurace do Dataverse TEST |
| Dataverse TEST | Managed řešení, cílové hodnoty a ověření funkčnosti |

DEV bude připojené ke Gitu. TEST v tomto scénáři ke Gitu nepřipojujte: dostává managed řešení přes pipeline. Pull z Gitu slouží k synchronizaci vývojových unmanaged změn; nenahrazuje nasazení do TEST.

Budeme používat **Azure DevOps Pipelines** s Power Platform Build Tools. Samostatná funkce **Pipelines in Power Platform** je jiná možnost nasazování; pro toto cvičení nepotřebujete její host environment. Azure DevOps Environment, které níže nazveme `pp-alm-test`, je zase záznam pro historii a schvalování nasazení v Azure DevOps, nikoli vaše Dataverse prostředí.

### 14.2 Ověřte licence, tenant a oprávnění

M365 E5 a Azure účet vám poskytují základ pro identitu a služby, ale **samotná M365 E5 ani Azure subscription nezajišťují oprávnění k aktivnímu používání aplikací a flow v Managed Environments**. Power Apps Developer Plan toto oprávnění pro běh prostředků také nezahrnuje. Pro toto cvičení se samostatným Power Automate flow je vhodné ověřit Power Automate Premium nebo dostupnou odpovídající trial licenci. Trial má vlastní omezení a dobu platnosti podle nabídky; její dostupnost závisí na vašem tenantu.

1. V [Microsoft 365 admin center](https://admin.microsoft.com/) otevřete svůj účet v Users → Active users → Licenses and apps a zkontrolujte skutečně přiřazené licence.
2. V [Power Platform admin center](https://admin.powerplatform.microsoft.com/) ověřte, že DEV a TEST mají Dataverse databázi.
3. V [Microsoft Entra admin center](https://entra.microsoft.com/) si poznamenejte Directory/Tenant ID tenantu, ve kterém máte Power Platform. U svého Azure účtu zkontrolujte aktivní directory. Stejný e-mail sám o sobě neprokazuje shodný tenant.
4. Azure DevOps organizaci připojte ke stejnému Entra tenantu. Nativní Dataverse Git integrace nepodporuje propojení napříč tenanty.
5. Pro zapnutí Managed Environments potřebujete tenantovou roli Power Platform Administrator nebo Dynamics 365 Administrator. Pro nastavení Git vazby potřebujete v Dataverse DEV roli System Administrator. Jsou to různá oprávnění.
6. V Azure DevOps ověřte přístup **Basic** a právo číst a commitovat do repozitáře, typicky členství v **Contributors**. Stakeholder pro práci s tímto repozitářem nestačí. Potřebujete také oprávnění spravovat pipeline, service connections a instalovat rozšíření do organizace.

Podle dokumentace Dataverse Git integrace mají být vývojová i cílová prostředí zapnutá jako Managed Environments. Jakmile máte vyřešené licence, v admin center otevřete Manage → Environments → … vedle DEV → Enable Managed Environments. Totéž proveďte pro TEST. Pro první pokus ponechte ostatní nastavení na výchozích hodnotách.

Managed Environment je nastavení prostředí; managed solution je typ nasazovaného balíčku. DEV tedy bude **Managed Environment s unmanaged solution** a TEST **Managed Environment s managed solution**.

Výstup: znáte tenant, máte odpovídající licenci a oprávnění a obě prostředí jsou připravená.

Zdroje: [licence Managed Environments](https://learn.microsoft.com/en-us/power-platform/admin/managed-environment-licensing), [zapnutí a administrátorské role](https://learn.microsoft.com/en-us/power-platform/admin/managed-environment-enable), [předpoklady Git integrace](https://learn.microsoft.com/en-us/power-platform/alm/git-integration/connecting-to-azure-devops), [Git FAQ včetně tenantů](https://learn.microsoft.com/en-us/power-platform/alm/git-integration/faqs).

### 14.3 Vytvořte Azure DevOps organizaci, projekt a repozitář

1. Otevřete [Azure DevOps](https://dev.azure.com/) a přihlaste se pracovním účtem z M365 tenantu. Pokud organizaci ještě nemáte, vytvořte ji.
2. V Organization settings → Microsoft Entra ID ověřte připojenou directory. Pokud připojení chybí, použijte Connect directory a vyberte tenant Power Platform. U existující týmové organizace je změna directory samostatná migrace uživatelů; pro svůj lab můžete založit novou organizaci rovnou ve správném tenantu.
3. Zvolte New project: například `PowerPlatform-ALM-Lab`, Visibility **Private**, Version control **Git**. Work item process můžete ponechat výchozí.
4. V Repos vytvořte repozitář `pp-alm-native`, pokud chcete jiný název než automaticky vytvořený repozitář projektu. Inicializujte ho s README pomocí Initialize. Zcela prázdný repozitář ještě nemá potřebnou výchozí větev.
5. Ověřte, že se výchozí větev jmenuje `main`. Pokud používáte jiný název, upravte i YAML a podmínku nasazení níže.
6. V Repos → Branches vytvořte z `main` pracovní větev `dev/radim`. K ní připojíme DEV; do `main` půjdou změny přes pull request.
7. V Organization settings → Users a Project settings → Permissions zkontrolujte svůj Basic přístup a práva k repozitáři.

Azure účet neznamená, že už máte Azure DevOps organizaci, projekt nebo kapacitu pro spouštění pipeline. Git repozitář a výpočetní agent pro pipeline jsou dvě samostatné části.

Výstup: existují Azure DevOps organizace ve správném tenantu, soukromý projekt, inicializovaný repozitář a větve `main` a `dev/radim`.

Zdroj: [připojení organizace k Entra ID](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/connect-organization-to-azure-ad?view=azure-devops).

### 14.4 Připojte existující solution v DEV k Azure Repos

1. V [Power Apps maker portálu](https://make.powerapps.com/) vyberte **DEV**. Stejné Solutions jsou dostupné i v Power Automate.
2. Otevřete své custom unmanaged řešení `PPAlmLab`. Pokud je stále prázdné, nejprve do něj přidejte flow a dvě proměnné z bodů 4–6.
3. V Source control zvolte **Connect to Git**; volba může být dostupná i na seznamu Solutions.
4. Pro tento lab vyberte **Solution binding**: verzovat chcete právě toto řešení. **Environment binding** je vhodné pro dedikované vývojové prostředí, kde chcete automaticky verzovat všechny unmanaged custom solutions. Microsoft jej doporučuje jako obecný výchozí přístup; zde volím solution binding kvůli rozsahu jednoho cvičení.
5. Vyberte Azure DevOps organizaci, projekt a repozitář `pp-alm-native`.
6. Vyberte větev **`dev/radim`** a Git folder **`native`**. Tato složka bude kořen nativních zdrojů. Pipeline níže předpokládá právě tuto cestu.
7. Zvolte Connect. Pokud průvodce nejprve nastaví solution binding pro prostředí, dokončete také vazbu konkrétního řešení přes … → Connect to Git na jeho řádku; připojení prostředí samo ještě nemusí znamenat připojení konkrétní solution.
8. Otevřete Source control a ověřte repo, větev a složku. Použijte Refresh, prohlédněte Changes a proveďte první **Commit** s popisem `Initial native source for PPAlmLab`.
9. V Azure Repos přepněte na `dev/radim` a ověřte vznik commitu a souborů pod `native/`.

Default Solution ani Common Data Service Default Solution tímto způsobem nepřipojujte. U solution binding respektujte omezení sdílených komponent: jeden objekt musí mít jediné místo v source control. Pozdější víceřešení návrh proto promyslete podle závislostí a zvolené vazby.

Výstup: změny vašeho řešení se dostanou do Azure Repos přímo z maker portálu, bez ručního exportu a unpacku pro každý commit.

Zdroj: [environment versus solution binding a připojení](https://learn.microsoft.com/en-us/power-platform/alm/git-integration/connecting-to-git).

### 14.5 Zkontrolujte zdroje a přidejte konfiguraci

Nativní integrace vytváří YAML manifesty i další soubory podle typu komponent. Soubory a jejich složky nechte vygenerovat Dataverse; nevytvářejte prázdné náhražky manifestů. Příklad uspořádání pro naše nastavení:

| Cesta od kořene repozitáře | Účel | Verzovat |
| --- | --- | --- |
| `native/solutions/PPAlmLab/solution.yml` | Metadata a verze řešení | Ano |
| `native/solutions/PPAlmLab/solutioncomponents.yml` a další manifesty v této složce | Seznamy komponent a závislostí | Ano |
| `native/publishers/AlmLabPublisher/publisher.yml` | Publisher | Ano |
| `native/modernflows/` | Zdrojové soubory cloud flow, pokud je integrace takto vytvoří | Ano |
| `native/environmentvariabledefinitions/` | Definice proměnných prostředí | Ano |
| Další složky vytvořené v `native/` | Ostatní komponenty řešení | Ano |
| `config/dev.settings.json` | Netajné DEV hodnoty jako záznam požadované konfigurace | Ano |
| `config/test.settings.json` | Netajné hodnoty použité při importu do TEST | Ano |
| `azure-pipelines.yml` | Definice sestavení a nasazení | Ano |
| `README.md`, `.gitignore` | Postup a pravidla nového repozitáře | Ano |
| `artifacts/` | Lokální sestavené ZIPy | Ne |

Tento layout nahrazuje zdrojovou složku `src/PPAlmLab/` pro **nový nativní repozitář**. V původním ručním cvičení se tato složka používá dál. Sestavení nativního řešení bude číst celý kořen `native/`, tedy i publisher a komponenty; samotná složka `native/solutions/PPAlmLab/` nestačí.

V Azure Repos přidejte do **`dev/radim`** soubory `config/dev.settings.json`, `config/test.settings.json`, README a `.gitignore`. Lze použít webový editor nebo nový lokální clone tohoto repozitáře. Do `.gitignore` přidejte:

```gitignore
/artifacts/
```

`config/dev.settings.json`:

```json
{
  "EnvironmentVariables": [
    { "SchemaName": "lab_EnvironmentName", "Value": "DEV" },
    { "SchemaName": "lab_MessagePrefix", "Value": "Vývojový pokus" }
  ],
  "ConnectionReferences": []
}
```

`config/test.settings.json`:

```json
{
  "EnvironmentVariables": [
    { "SchemaName": "lab_EnvironmentName", "Value": "TEST" },
    { "SchemaName": "lab_MessagePrefix", "Value": "Testovací pokus" }
  ],
  "ConnectionReferences": []
}
```

DEV JSON se v této pipeline neaplikuje: DEV už má své Current values. Je to verzovaný záznam konfigurace pro případ obnovení vývojového prostředí. TEST JSON se při každém nasazení skutečně použije.

Před commitem a v prvním diffu zkontrolujte také případné soubory s environment variable values. Nativní formát může hodnoty obsahovat; samotné připojení ke Gitu nezaručuje, že se DEV konfigurace v balíčku neobjeví. Zachovejte postup vyloučení Current values z řešení z bodu 6, ověřte obsah zdrojů a TEST hodnoty nastavujte deployment settings souborem. Ruční vymazání hodnot v Git souborech bez odpovídající změny v DEV by při dalším commitu mohlo být přepsáno.

Do README nového repozitáře napište Unique name, publisher prefix, role DEV/TEST, vazbu na `dev/radim` a složku `native/`, název pipeline a service connection. Postup bude: **změna v DEV → native Commit → PR do main → pack managed → artifact → import s TEST settings → ověření flow**. Přihlašovací údaje, PAT, client secret ani tokeny do README a JSON nedávejte. Connection reference obsahuje mapování, nikoli heslo ke konektoru.

Zdroj: [nativní YAML formát a složky](https://learn.microsoft.com/en-us/power-platform/alm/solution-source-control-yaml-format).

### 14.6 Vyzkoušejte nativní commit a pull

1. V DEV změňte flow, například přidejte do výsledné zprávy text `Native Git v1`. Uložte flow, proveďte potřebné publikování změn a ověřte běh v DEV.
2. V solution → Source control použijte Refresh. Prohlédněte Changes, zkontrolujte obsah a proveďte Commit s popisem změny. Ověřte jej v Azure Repos ve větvi `dev/radim`.
3. Pro nácvik opačného směru upravte v Git repozitáři podporovaný údaj, například popis řešení v manifestu podle jeho skutečné struktury. Nezkoušejte naslepo přepisovat interní definici flow; podporu přímých změn souborů posuzujte podle typu komponenty.
4. V maker portálu zvolte Check for updates. Prohlédněte Updates a použijte Pull. Ověřte změněný popis a znovu spusťte flow.
5. Pokud jsou Conflicts, otevřete každý konflikt a rozhodněte mezi Keep existing changes z DEV a Accept incoming changes z Gitu. Teprve po jejich vyřešení pokračujte pullem nebo commitem.

Commit v maker portálu uloží změny do vzdáleného repozitáře; za ním nepotřebujete samostatný lokální `git push`. Webové úpravy README nebo pipeline v Azure Repos zase nepotřebují ruční export řešení.

Výstup: rozumíte oběma směrům synchronizace a vidíte historii změn. TEST se tímto krokem ještě nemění.

Zdroj: [Source control operations](https://learn.microsoft.com/en-us/power-platform/alm/git-integration/source-control-operations).

### 14.7 Připravte Build Tools a agenta pro pipeline

1. Do Azure DevOps organizace nainstalujte rozšíření Microsoft **Power Platform Build Tools** z [Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=microsoft-IsvExpTools.PowerPlatform-BuildTools). Používejte tasky generace `@2`.
2. V Organization settings → Pipelines → Parallel jobs ověřte dostupnost Microsoft-hosted agenta. YAML níže používá `windows-latest`.
3. Pokud nová organizace nemá přidělený Microsoft-hosted parallel job, požádejte o bezplatnou kapacitu postupem z dokumentace Azure DevOps. Přidělení nemusí být okamžité. Chyba `No hosted parallelism has been purchased or granted` se týká agenta, nikoli Power Platform přihlášení.
4. Alternativou je vlastní Windows self-hosted agent. V Project settings → Agent pools / příslušné organizační správě poolů vytvořte pool například `PP-Lab`, stáhněte a zaregistrujte agenta podle průvodce a ponechte jej online. Na stroji zajistěte Git, PowerShell 7 a přístup k Azure DevOps, NuGet a Dataverse. Pro tento malý lab může jít o váš počítač; Azure VM není nutnou podmínkou.
5. Pro self-hosted variantu nahraďte v YAML **celý** blok `pool` tímto:

```yaml
pool:
  name: PP-Lab
```

Dočasný údaj použitý při registraci agenta není obsah repozitáře. Pipeline níže si instaluje .NET SDK i CLI sama. Nemusí používat `pac` z vašeho lokálního PowerShellu.

Výstup: organizace má Build Tools a alespoň jeden dostupný agent.

Zdroje: [Power Platform Build Tools](https://learn.microsoft.com/en-us/power-platform/alm/devops-build-tools), [parallel jobs a žádost o kapacitu](https://learn.microsoft.com/en-us/azure/devops/pipelines/licensing/concurrent-jobs?view=azure-devops), [Windows self-hosted agent](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/windows-agent?view=azure-devops).

### 14.8 Připravte aplikační identitu pro nasazení do TEST

Lidský účet používá maker portál a nativní Git. Pipeline bude importovat řešení pod samostatnou aplikační identitou. Tato dvě přihlášení nejsou totožná.

1. V Microsoft Entra ID → App registrations → New registration vytvořte například `pp-alm-deploy-test` jako aplikaci pro svůj tenant. Redirect URI pro toto nasazování nepotřebujete.
2. Poznamenejte **Application (client) ID** a **Directory (tenant) ID**. Nezaměňujte client ID s Object ID aplikace nebo enterprise application.
3. V Power Platform admin center otevřete Manage → Environments → **TEST** → Settings → Users + permissions → Application users.
4. Zvolte New app user → Add an app a vyberte vytvořenou registraci podle client ID.
5. Vyberte Business Unit a doplňte další povinné údaje průvodce. Pro první izolovaný lab přiřaďte roli **System Administrator**, potvrďte a vytvořte uživatele. Pro týmový provoz následně navrhněte roli s oprávněními potřebnými k importu a práci s danými komponentami.
6. Ověřte, že application user v TEST je aktivní a má správnou roli.

V této pipeline Build pracuje jen se soubory a do DEV se nepřihlašuje, proto deploy aplikaci v DEV nepotřebuje. Aplikační uživatel pro import také sám nepřiděluje licence lidem ani přístup k SharePoint connection.

Výstup: TEST zná aplikační identitu, která bude provádět import.

Zdroj: [správa a vytvoření application user](https://learn.microsoft.com/en-us/power-platform/admin/manage-application-users).

### 14.9 Vytvořte Power Platform service connection s federací

1. V Azure DevOps projektu otevřete Project settings → Service connections → New service connection → **Power Platform**. Pro tento import potřebujete Power Platform připojení; Azure Resource Manager připojení k subscription má jiný účel.
2. Vyberte **Workload Identity federation**. Některé verze UI tuto volbu označují `(preview)`.
3. Vyplňte Server URL skutečnou Dataverse URL **TEST**, Tenant ID a Application ID z předchozího kroku.
4. Připojení pojmenujte přesně **`PP-TEST-WIF`** a uložte. Pokud UI nabízí Verify and save ještě před nastavením federace, nejprve použijte uložení bez ověření, je-li dostupné; ověření dokončíte po následujících krocích.
5. Ze stránky service connection zkopírujte přesné hodnoty **Issuer** a **Subject identifier**. Neodvozujte je ručně z názvu organizace. Přesná shoda je podmínkou přihlášení.
6. V Entra ID otevřete aplikaci `pp-alm-deploy-test` → Certificates & secrets → Federated credentials → Add credential → **Other issuer**.
7. Vložte Issuer a Subject identifier z Azure DevOps, zkontrolujte audience `api://AzureADTokenExchange` pro tento typ federace a zadejte jméno, například `azdo-pp-test`. Credential uložte.
8. Vraťte se do service connection a ověřte ji, pokud UI nabízí Verify. Skutečný přístup do Dataverse následně prověří i task WhoAmI v pipeline.
9. V oprávněních service connection povolte použití vytvořené pipeline. První spuštění může nabídnout tlačítko Permit/Authorize resources. Pro lab stačí autorizovat konkrétní pipeline.

Federace umožní pipeline získat krátkodobý token bez ukládání client secret. Ve výukovém postupu nepotřebujete Key Vault ani běžnou Azure Resource Manager service connection. Pokud WIF v nabídce není, nejprve ověřte nainstalované a aktuální Build Tools. Alternativní SPN připojení s client secret je podporované, ale secret patří do zabezpečeného pole service connection a vyžaduje správu expirace; do tohoto YAML ani do `config/*.json` ho nedávejte.

Výstup: existuje `PP-TEST-WIF`, federated credential a application user v TEST se shodným client ID.

Zdroje: [doporučená autentizace Build Tools](https://learn.microsoft.com/en-us/power-platform/alm/devops-build-tools), [Microsoft postup vytvoření Power Platform WIF připojení, kroky 1–2](https://learn.microsoft.com/en-us/power-platform/admin/unified-experience/tutorial-scheduled-copy-with-data-import). Druhý odkaz dále řeší finance and operations; jeho ERP API oprávnění a kroky kopírování prostředí k našemu importu nepřebírejte.

### 14.10 Připravte schvalování nasazení

1. V Azure DevOps otevřete Pipelines → Environments → New environment.
2. Název nastavte **`pp-alm-test`**, resource zvolte None. Dataverse URL je uložená v service connection, toto není registrace VM nebo Kubernetes clusteru.
3. V tomto Azure DevOps Environment otevřete Approvals and checks → Add → Approvals.
4. Pro samostatný lab nastavte sebe jako approvera a povolte schválení vlastního běhu. V týmu určete jiného schvalovatele. Zkontrolujte i oprávnění pipeline k použití environment.
5. Uložte. Kontrola čeká před vstupem do deployment stage; samotný Build se může dokončit bez schválení.

Tím si nacvičíte continuous delivery: build a příprava nasazení proběhnou automaticky, TEST nasazení spustíte schválením. Později můžete v TEST approval zrušit a nasazovat automaticky po mergi; pro PROD obvykle ponecháte samostatnou kontrolu.

Zdroj: [Azure Pipelines approvals and checks](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops).

### 14.11 Přidejte kompletní YAML pipeline

V `dev/radim` vytvořte v kořeni repozitáře soubor **`azure-pipelines.yml`**. Příklad předpokládá právě jedno řešení pod `native/solutions/`, výše uvedenou strukturu, service connection `PP-TEST-WIF` a Azure DevOps Environment `pp-alm-test`. Pokud máte jiný Unique name nebo Schema names, nahraďte je v ukázce i JSON souborech.

Nativní YAML zdroje umí zabalit PAC od verze **2.4.1**. Ukázka připíná vydaný `Microsoft.PowerApps.CLI.Tool` **2.12.2** a používá .NET SDK **10.x**, které tato verze vyžaduje. Aktualizaci CLI později udělejte samostatným commitem a ověřte sestavení; není nutné sledovat „latest“ v každém buildu. Pro native pack není potřeba dřívější dvojitý export Managed/Unmanaged a unpack Both.

```yaml
trigger:
  batch: true
  branches:
    include:
      - main
      - dev/*

pool:
  vmImage: windows-latest

variables:
  SolutionName: PPAlmLab
  NativeRoot: '$(Build.SourcesDirectory)/native'
  PacVersion: '2.12.2'
  TestServiceConnection: PP-TEST-WIF

stages:
  - stage: Build
    displayName: Build managed solution from Git
    jobs:
      - job: Pack
        timeoutInMinutes: 60
        steps:
          - checkout: self
            clean: true

          - task: UseDotNet@2
            displayName: Install .NET SDK
            inputs:
              packageType: sdk
              version: 10.x

          - pwsh: |
              $ErrorActionPreference = 'Stop'
              $sourceRoot = '$(NativeRoot)'
              $solutionName = '$(SolutionName)'
              $manifest = Join-Path $sourceRoot "solutions/$solutionName/solution.yml"
              if (-not (Test-Path $manifest)) {
                throw "Native manifest missing: $manifest"
              }
              $manifests = @(Get-ChildItem (Join-Path $sourceRoot 'solutions') `
                -Filter solution.yml -Recurse)
              if ($manifests.Count -ne 1) {
                throw 'This lab pipeline expects exactly one native solution.'
              }
              if (-not (Test-Path (Join-Path $sourceRoot 'publishers'))) {
                throw 'Native publishers folder is missing.'
              }

              $settingsPath = '$(Build.SourcesDirectory)/config/test.settings.json'
              $settings = Get-Content $settingsPath -Raw | ConvertFrom-Json
              $names = @($settings.EnvironmentVariables | ForEach-Object { $_.SchemaName })
              foreach ($required in @('lab_EnvironmentName', 'lab_MessagePrefix')) {
                if ($required -notin $names) {
                  throw "TEST configuration is missing $required"
                }
              }
              $targetName = @($settings.EnvironmentVariables | Where-Object {
                $_.SchemaName -eq 'lab_EnvironmentName'
              })
              if ($targetName.Count -ne 1 -or $targetName[0].Value -ne 'TEST') {
                throw 'TEST deployment must use exactly one TEST environment name.'
              }

              $toolDir = Join-Path $env:AGENT_TEMPDIRECTORY "pac-$env:BUILD_BUILDID"
              if (Test-Path $toolDir) { Remove-Item $toolDir -Recurse -Force }
              dotnet tool install Microsoft.PowerApps.CLI.Tool `
                --tool-path $toolDir --version '$(PacVersion)' `
                --source 'https://api.nuget.org/v3/index.json'
              if ($LASTEXITCODE -ne 0) { throw 'PAC installation failed.' }
              $pac = Join-Path $toolDir 'pac.exe'
              & $pac help
              if ($LASTEXITCODE -ne 0) { throw 'PAC could not start.' }

              $artifactDir = '$(Build.ArtifactStagingDirectory)/solution'
              New-Item $artifactDir -ItemType Directory -Force | Out-Null
              $zipPath = Join-Path $artifactDir "${solutionName}_managed.zip"
              & $pac solution pack --folder $sourceRoot `
                --zipfile $zipPath --packagetype Managed
              if ($LASTEXITCODE -ne 0) { throw 'Native solution pack failed.' }

              Add-Type -AssemblyName System.IO.Compression.FileSystem
              $archive = [System.IO.Compression.ZipFile]::OpenRead($zipPath)
              try {
                $entry = $archive.GetEntry('solution.xml')
                if ($null -eq $entry) { throw 'Package is missing solution.xml.' }
                $reader = [System.IO.StreamReader]::new($entry.Open())
                try { [xml]$solutionXml = $reader.ReadToEnd() }
                finally { $reader.Dispose() }
              }
              finally { $archive.Dispose() }
              $info = $solutionXml.ImportExportXml.SolutionManifest
              if ([string]$info.Managed -ne '1' -or
                  [string]$info.UniqueName -ne $solutionName) {
                throw 'Package is not the expected managed solution.'
              }

              Copy-Item $settingsPath (Join-Path $artifactDir 'test.settings.json')
              [ordered]@{
                solution = $solutionName
                solutionVersion = [string]$info.Version
                commit = $env:BUILD_SOURCEVERSION
                branch = $env:BUILD_SOURCEBRANCH
                buildId = $env:BUILD_BUILDID
                pacVersion = '$(PacVersion)'
                zipSha256 = (Get-FileHash $zipPath -Algorithm SHA256).Hash
              } | ConvertTo-Json | Set-Content `
                (Join-Path $artifactDir 'provenance.json') -Encoding utf8
            displayName: Validate configuration and pack native source

          - publish: '$(Build.ArtifactStagingDirectory)/solution'
            artifact: solution
            displayName: Publish managed ZIP, TEST settings and provenance

  - stage: Deploy_TEST
    displayName: Deploy managed solution to TEST
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'), ne(variables['Build.Reason'], 'PullRequest'))
    jobs:
      - deployment: Import_TEST
        timeoutInMinutes: 60
        environment: pp-alm-test
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: none
                - download: none
                - download: current
                  artifact: solution

                - task: PowerPlatformToolInstaller@2
                  displayName: Install Power Platform Build Tools
                  inputs:
                    DefaultVersion: true

                - task: PowerPlatformWhoAmi@2
                  displayName: Check TEST application identity
                  inputs:
                    authenticationType: PowerPlatformSPN
                    PowerPlatformSPN: '$(TestServiceConnection)'

                - task: PowerPlatformImportSolution@2
                  displayName: Import managed solution with TEST configuration
                  inputs:
                    authenticationType: PowerPlatformSPN
                    PowerPlatformSPN: '$(TestServiceConnection)'
                    SolutionInputFile: '$(Pipeline.Workspace)/solution/$(SolutionName)_managed.zip'
                    UseDeploymentSettingsFile: true
                    DeploymentSettingsFile: '$(Pipeline.Workspace)/solution/test.settings.json'
                    AsyncOperation: true
                    MaxAsyncWaitTime: '45'
                    PublishWorkflows: true
                    HoldingSolution: false
                    OverwriteUnmanagedCustomizations: false
```

Build si vezme konkrétní checkout. **V pipeline není Export Solution z živého DEV**: nezahrne tedy omylem necommitnuté změny. Artifact obsahuje ZIP, TEST konfiguraci a `provenance.json` s commitem, verzí a kontrolním součtem. Deploy stáhne artifact stejného běhu; znovu nesestavuje ani nečte později změněný `main`.

Build poběží na `main` i `dev/*`. Deploy poběží jen pro `main` po úspěšném buildu; PR a pracovní větev do TEST neimportují. První schvalování nastavujete v Azure DevOps Environment, nikoli vložením hesla nebo approvera do YAML.

Úspěšný pack ještě neprokazuje úplnost všech komponent nebo funkčnost flow. Prohlédněte také warnings v logu, import a výsledek testu. V prvním labu jsou kontroly omezené na sestavení, typ balíčku, očekávané jméno a základní konfiguraci; checker a automatické funkční testy přidáte později.

Zdroje: [PAC solution pack](https://learn.microsoft.com/en-us/power-platform/developer/cli/reference/solution#pac-solution-pack), [CLI.Tool 2.12.2 a .NET](https://www.nuget.org/packages/Microsoft.PowerApps.CLI.Tool/2.12.2), [parametry Build Tools tasků](https://learn.microsoft.com/en-us/power-platform/alm/devops-build-tool-tasks).

### 14.12 Založte pipeline, ověřte Build a zapněte PR pravidla

1. Commitněte `azure-pipelines.yml` a oba konfigurační soubory do `dev/radim`. V této větvi již musí být také první native commit pod `native/`.
2. V Azure DevOps zvolte Pipelines → New pipeline → Azure Repos Git → `pp-alm-native` → Existing Azure Pipelines YAML file.
3. Vyberte větev `dev/radim` a `/azure-pipelines.yml`. Pipeline pojmenujte například `PPAlmLab-CI-CD`.
4. Spusťte ji nejprve ručně nad `dev/radim`. Autorizujte potřebné resources. Build má uspět a vytvořit artifact `solution`; Deploy_TEST musí být **Skipped**.
5. Otevřete artifact, ověřte ZIP, `test.settings.json` a `provenance.json`. Především zkontrolujte skutečný Unique name, verzi a TEST hodnoty.
6. Po prvním úspěšném buildu otevřete Repos → Branches → `main` → … → Branch policies.
7. Nastavte Build validation → Add: vyberte `PPAlmLab-CI-CD`, Trigger **Automatic**, Policy requirement **Required** a expiraci při změně cílové větve. Pro první lab ponechte bez path filtru, aby se kontrolovala i konfigurace a YAML.
8. Zapněte vyřešení komentářů. Reviewer policy nastavte podle počtu lidí: v samostatném labu můžete kontrolu provést sám bez povinného cizího review; v týmu vyžadujte alespoň jednoho dalšího reviewera.
9. Ověřte, že běžný vývojový účet nepoužívá Bypass policies k přímému přepisování `main`. Změny posílejte PR.

**V Azure Repos se PR build validation nastavuje přes branch policies.** Samotný YAML klíč `pr:` ji pro Azure Repos nenastaví. CI trigger na `dev/*` a PR validation mohou při otevřeném PR vyvolat dva buildy; pro první pokus je to v pořádku. Později lze větevní CI omezit a ponechat PR kontrolu.

Výstup: build funguje bez připojení k DEV a `main` vyžaduje úspěšnou validaci před sloučením.

Zdroje: [Azure Repos PR triggers](https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/azure-repos-git?view=azure-devops#pr-triggers), [Build validation policy](https://learn.microsoft.com/en-us/azure/devops/repos/git/branch-policies?view=azure-devops#build-validation).

### 14.13 Proveďte první PR a nasazení do TEST

1. V Repos → Pull requests vytvořte PR z `dev/radim` do `main`. Popište, co flow dělá, jeho verzi a očekávané DEV/TEST hodnoty.
2. Prohlédněte Files: native zdroje, publisher, definice proměnných, JSON konfiguraci a YAML. Ověřte, že nejsou přibalené přihlašovací údaje.
3. Nechte doběhnout required Build validation. V PR běhu má Deploy_TEST zůstat Skipped.
4. Dokončete review a Complete. **Nemažte source branch `dev/radim`**, protože k ní je připojené DEV.
5. Merge do `main` vyvolá další běh pipeline. Ten sestaví artifact z výsledného main commitu a bude čekat na approval pro `pp-alm-test`.
6. V přehledu běhu otevřete čekající Deploy_TEST, zkontrolujte commit/verzi/artifact a potvrďte approval.
7. V logu zkontrolujte WhoAmI: má jít o správnou aplikační identitu a TEST. Následně má uspět managed import s deployment settings.
8. V Power Automate přepněte na TEST → Solutions → `PPAlmLab`. Ověřte Managed, verzi, obě Current values a stav flow.
9. Spusťte flow. Výsledek má obsahovat `Prostredi=TEST`, `Testovací pokus` a vaši změnu `Native Git v1`. V DEV má stále být DEV a jeho vlastní zpráva.
10. Pokud flow v TEST nevidíte nebo nemůžete spustit, ověřte přístup/co-owner či run-only oprávnění lidského testovacího účtu. Aplikační identita importu a uživatel, který flow testuje, mají odlišné role.

Je-li v TEST již managed `PPAlmLab` z prvního cvičení, zachovejte Unique name a publisher a nasaďte vyšší verzi. Pokud je v TEST originating unmanaged varianta stejného řešení, nejprve vyřešte tento konflikt podle původního plánu; nevytvářejte druhé řešení s náhodným názvem jen kvůli obejití importu.

Import v této ukázce provádí běžnou aktualizaci. Odebrání komponenty ze zdrojů a následný Update nemusí odstranit komponentu z TEST. Pro nácvik odstraňování navrhněte samostatný **Upgrade** postup, například import holding solution a následný Apply Solution Upgrade. Před takovou změnou posuďte dopad na data a závislosti.

Výstup: merge do `main` prokazatelně nasadil managed solution a správnou konfiguraci přes Azure DevOps.

### 14.14 Nacvičte další verzi a konfiguraci bez změny flow

**Další verze řešení:**

1. Po mergi aktualizujte `dev/radim` změnami z `main`. Můžete v Azure Repos vytvořit opačný synchronizační PR `main` → `dev/radim` a dokončit jej bez smazání `main`; případně v lokálním clone použijte `git fetch origin`, `git switch dev/radim`, `git merge origin/main`, `git push origin dev/radim`. Nejprve vyřešte případné Git konflikty.
2. V DEV → Source control zkontrolujte updates/conflicts a proveďte Pull, pokud jsou změny dostupné. Vazba na `dev/radim` zůstává stejná.
3. V DEV změňte flow, otestujte jej a ve Settings solution zvyšte verzi, například z `1.0.1.0` na `1.0.2.0`, podle skutečně aktuální verze.
4. Commitněte flow i metadata řešení. Zkontrolujte, že nová verze je v `solution.yml` skutečně zachycená. Git commit ani `Build.BuildId` nezvyšují solution version automaticky.
5. Proveďte PR do `main`, build, approval a import. Ověřte v TEST novou verzi a chování.

Nativní vazba je na jednu konkrétní větev. Pro tento dvouprostředí lab proto používáme trvalou pracovní větev. Nepředpokládejte, že lokální `git switch` přepne i Dataverse. Přechod native binding na jinou větev vyžaduje odpojení a opětovné připojení podle podporovaného postupu. Návrh izolovaných vývojových prostředí a feature branches rozšiřte až po dokončení tohoto základu.

**Pouze změna TEST konfigurace:**

1. V `dev/radim` upravte `config/test.settings.json`, například MessagePrefix na `Testovací pokus – konfigurace 2`.
2. Po commitu a PR do `main` pipeline znovu sestaví a importuje tentýž zdrojový stav řešení s novou TEST konfigurací. Pro tento lab to názorně ukáže, že změna hodnot není změnou logiky flow.
3. Ověřte novou Current value. Pokud běh používá starou hodnotu, vypněte a znovu zapněte flow a test opakujte.
4. DEV musí nadále používat své hodnoty. Ruční změnu TEST hodnoty mimo Git při dalším nasazení může přepsat verzovaný TEST JSON.

Později oddělte konfiguraci od importu řešení a přidejte samostatnou konfigurační pipeline, pokud to potřebujete. Cvičný YAML nyní záměrně používá jeden čitelný postup.

### 14.15 Přidejte SharePoint a přibližte postup týmovému vývoji

Nejprve dokončete jeden funkční cyklus bez konektorů. Pak navazujte bodem 13:

1. Přidejte v DEV flow se SharePoint akcí a solution-aware connection reference. Definice se dostane do native commitu spolu s flow.
2. V TEST předem vytvořte cílové SharePoint prostředky a autorizované připojení. Import solution nevytvoří SharePoint list ani nepřenáší heslo/OAuth souhlas k připojení.
3. Z aktuálně sestaveného ZIPu můžete vygenerovat šablonu příkazem `pac solution create-settings --solution-zip .\artifacts\PPAlmLab_managed.zip --settings-file .\config\test.template.json`. Načtěte skutečná Schema names a logical name reference; šablonu poté doplňte do `test.settings.json`.
4. Přidejte TEST Site URL, GUID cílového listu a skutečné mapování connection reference na connection v TEST. Typický záznam je `LogicalName`, `ConnectionId`, `ConnectorId`; konkrétní hodnoty převezměte ze šablony a cílového připojení.
5. Ověřte, že importující identita a vlastník flow smějí dané connection použít. Způsob sdílení a podporu service principal posuďte podle konkrétního konektoru. Samotná role System Administrator v Dataverse nezajišťuje přihlášení do SharePointu.
6. Přes PR nasaďte a ověřte, že TEST zapisuje výhradně do TEST seznamu. U flow s premium konektory ověřte také licenci podle vlastníka a způsobu spuštění. Je-li vlastníkem service principal, řešte Process/per-flow licenci, licencovanou flow group nebo podporovaný designated licensed user podle aktuálních pravidel. Licence pro aktivní použití Managed Environments a licence pro tento způsob běhu flow mají odlišný účel.

Po funkčním DEV/TEST cyklu rozšiřte návrh postupně:

| Rozšíření | Praktická změna |
| --- | --- |
| PROD | Další Dataverse prostředí, vlastní service connection, vlastní configuration a approval |
| Stejný release | Do TEST a PROD posílejte stejný již ověřený managed ZIP; mezi prostředími jej znovu nesestavujte |
| Izolace vývoje | Samostatná vývojová prostředí a promyšlené Git binding/branching podle aktuálně podporovaného modelu |
| Kvalita | Solution checker pro sestavený artifact a automatické testy důležitých scénářů |
| Verze a dohledatelnost | Verze solution, Git commit/tag a build/run propojené v release záznamu |
| Upgrade | Řízené odstranění komponent, závislosti a migrace dat |
| Provoz flow | Přístup testerů, vlastnictví, connections a licencování běhu |
| Souběžná nasazení | Serializace nasazení do jednoho cíle, například exclusive lock v Azure DevOps checks |

Build agent v ukázce sestavuje balíček ze zdrojů. Samostatné Dataverse BUILD prostředí přidejte, pokud vaše komponenty nebo testy vyžadují ověřovací import či další operace v Dataverse; pro tento jednoduchý flow není nezbytnou součástí pack kroku.

Zdroje: [deployment settings a connection references](https://learn.microsoft.com/en-us/power-platform/alm/conn-ref-env-variables-build-tools), [service principal owned flows a licence](https://learn.microsoft.com/en-us/power-automate/service-principal-support).

### 14.16 Co doložit a jak řešit typické chyby

| Důkaz | Očekávaný výsledek |
| --- | --- |
| Git connection v DEV | Správná organizace, repo, `dev/radim` a `native/` |
| Azure Repos historie | Alespoň dva native commity a viditelný diff flow/verze |
| Pull request | Validace před mergem do `main` |
| Build artifact | Managed ZIP, TEST JSON a provenance ke konkrétnímu commitu |
| Deployment | Úspěšné WhoAmI a import přes `PP-TEST-WIF`; historie v `pp-alm-test` |
| TEST solution | Managed a očekávaná verze |
| Flow běhy | Odlišné DEV/TEST hodnoty a stejná nová logika |
| Konfigurační změna | TEST hodnota změněná přes Git bez změny flow |

| Projev | Co ověřit |
| --- | --- |
| Chybí Connect to Git | Managed Environment, Dataverse, System Administrator, custom unmanaged solution a dostupnost funkce |
| V seznamu není Azure DevOps organizace | Stejný Entra tenant, pracovní identita, Basic přístup a repo oprávnění |
| Nelze vybrat branch | Inicializovaný repozitář a existující výchozí větev |
| Commit do main odmítnut | Branch policies; DEV připojte k pracovní větvi a použijte PR |
| Git ukazuje jinou větev než lokální clone | Dataverse binding je samostatné nastavení |
| Pipeline čeká / chyba hosted parallelism | Dostupný parallel job nebo online self-hosted agent |
| Unknown task PowerPlatform… | Instalované Build Tools v dané Azure DevOps organizaci, tasky `@2` |
| Chybí .NET / CLI se nespustí | UseDotNet krok, .NET 10 pro připnutou CLI.Tool 2.12.2 a úspěšná NuGet instalace |
| Pack hledá Customizations.xml | PAC podporující native YAML a správný kořen `native/` se `solutions/` a `publishers/` |
| Pack hlásí více řešení | Tato ukázka podporuje jedno řešení; upravte vícesolution sestavení s explicitním výběrem v SolutionPackager |
| PR nevyvolá kontrolu | Required Build validation na cílové větvi main, nikoli jen `pr:` v YAML |
| Deploy je Skipped | PR nebo pracovní větev; pro ně je to očekávané |
| Deploy čeká | Approval/checks nebo autorizace environment/service connection |
| WIF chyba AADSTS70021 či AADSTS700213 | Přesná shoda Issuer, Subject a audience; správný tenant a client ID |
| WhoAmI / import nemá práva | TEST application user, aktivní stav, role a správná Dataverse URL |
| Importovaný flow nelze spustit | Stav flow, oprávnění testera, vlastník, licence a u konektorů cílová connection |
| TEST používá DEV hodnoty | Přibalené Current values a skutečný obsah deployment settings artifactu |

YAML v tomto dokumentu je připravený vzor pro uvedený lab. Byla zkontrolována jeho YAML struktura a ukázky JSON konfigurace; pipeline nebyla spuštěna ve vašem Azure DevOps. Skutečný běh, přístup přes WIF a chování importovaného flow ověřte ve svém tenantu podle kroků 14.12–14.14.


## 15. Jak prokážete splnění zadání

| Co doložit | Konkrétní výsledek |
| --- | --- |
| Environments | DEV a TEST, v každém samostatná instance flow |
| Solutions | Unmanaged v DEV, managed v TEST |
| Environment variables | Stejná definice, odlišný výstup DEV/TEST |
| Git | XML/JSON zdroje ručního cvičení; pro bod 14 native zdroje v Azure Repos, commity a PR |
| Sestavení | Managed ZIP vytvořený příkazem pack z verzovaných souborů |
| Přenos konfigurace | test.settings.json použitý při CLI importu nebo import tasku pipeline |
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
| Pack managed selže | Pro starší XML postup obě varianty a unpack Both; pro native YAML v bodu 14 správný kořen zdrojů a podporovaná CLI |
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
