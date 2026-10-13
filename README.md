# Database Refresh Utility

**Česky** | [English](README.en.md)

Nástroj uloží vybrané nastavení TEST databáze před refreshem a potom je obnoví. **Samotný refresh provádí DBA.**

Postup: **Capture → refresh od DBA → Preflight → Restore → Validate**.

## 1. Co potřebuješ

- Windows PowerShell 5.1, SQL*Plus a Oracle 19c nebo novější.
- Funkční připojení přes SQL*Plus pod stejným Windows účtem, který spouští nástroj.
- Povolené spouštění PowerShell skriptů podle pravidel serveru. Další moduly ani Python nejsou potřeba.

## 2. Nastav připojení

Zkopíruj `config/database.example.json` jako `config/database.json`. Doplň účty a vyber jednu variantu připojení.

### Přes TNS alias

Alias `TESTDB` musí znát použitý Oracle klient, například přes `tnsnames.ora` nebo `TNS_ADMIN`. **Neuváděj `port` ani `serviceName`.**

```json
{
  "host": "TESTDB",
  "sqlplusPath": "sqlplus.exe",
  "timeoutSeconds": 300,
  "schemaOrder": ["APP1"],
  "users": {
    "APP1": {"username": "APP1", "password": "DOPLNTE_HESLO"}
  }
}
```

### Přímo přes server a službu

```json
{
  "host": "db-server.example.cz",
  "port": 1521,
  "serviceName": "testpdb.example.cz",
  "sqlplusPath": "sqlplus.exe",
  "schemaOrder": ["APP1"],
  "users": {
    "APP1": {"username": "APP1", "password": "DOPLNTE_HESLO"}
  }
}
```

`sqlplusPath` může být plná cesta ke klientovi. Výchozí hodnoty jsou `sqlplus.exe`, port `1521` a timeout `300` sekund pro jedno volání SQL*Plus. `host` přijímá i `//server:1521/sluzba` bez samostatného portu a služby.

## 3. Nastav operace pro každé schéma

Pro účet `APP1` vytvoř `config/schemas/APP1.json`. Název souboru, položka v `schemaOrder` a `username` musí odpovídat. `schemaOrder` musí obsahovat všechna nakonfigurovaná schémata právě jednou, v pořadí obnovy. Soubory `*.example.json` se ignorují.

Soubor schématu obsahuje pole `steps`. Kroky pro různé tabulky můžeš spojit do jednoho pole. Hesla patří jen do `database.json`.

### Jak se vybírají řádky

| Pole | Význam |
| --- | --- |
| `type` | Operace, kterou chceš provést. |
| `table` | Tabulka ve schématu přihlášeného účtu. |
| `key` | Sloupce pro výběr nebo identifikaci řádků. Nikdy se automaticky nepřebírá primární klíč. |
| `match` | Pole podmínek; každý objekt obsahuje přesně sloupce z `key`. |
| `allRows` | Hodnota `true` vybírá všechny řádky. |
| `columns` | Sloupce, jejichž původní hodnoty obnoví `restoreRows`. |
| `set` | Pevné hodnoty, které nastaví `update`. |

Vždy použij **buď `match`, nebo `"allRows": true`**. Uvnitř jednoho objektu `match` platí AND, mezi objekty OR. Prázdný výběr je v pořádku; prázdné pole `match` je chyba.

**Unikátnost záleží na operaci:**

- `restoreRows` a `insert`: každá kombinace `key` musí ve vybraných datech určit nejvýše jeden řádek a nesmí obsahovat NULL. Duplicity mimo `match` nevadí. Při Capture s `allRows` se kontroluje celá tabulka.
- `backupTable`, `update` a `delete`: podmínka smí vybrat více řádků. `null` a `""` znamenají `IS NULL`.

Sloupce `key` nesmějí být v `columns` ani `set`. Filtry podporují rovnost, nikoli vlastní SQL, LIKE nebo rozsahy. Hodnoty jsou řetězce, čísla nebo `null`; například příznak zapisuj jako `0`/`1`, ne `false`/`true`. Názvy tabulek a sloupců se převádějí na velká písmena; quoted identifiers se nepodporují.

### Přehled operací

| Operace | Co dělá | `key` s `match` | `key` s `allRows` |
| --- | --- | --- | --- |
| `backupTable` | Pouze záloha; nic automaticky neobnovuje. | Povinný | Bez `key` |
| `restoreRows` | Vrátí původní hodnoty zadaných `columns`. | Povinný | Povinný |
| `replaceTable` | Nahradí celý obsah tabulky. | Nepodporuje | Bez `key` |
| `insert` | Doplní chybějící zachycené řádky. | Povinný | Povinný |
| `update` | Nastaví hodnoty ze `set` všem vybraným řádkům. | Povinný | Povinný |
| `delete` | Smaže všechny vybrané řádky. | Povinný | Bez `key` |

## 4. Příklady konfigurace

Každý příklad je samostatný obsah souboru schématu. Názvy tabulek, sloupců a hodnoty nahraď vlastními.

### backupTable — pouze záloha

Vytvoří CSV a INSERT SQL. Preflight, Restore a Validate tuto tabulku přeskočí.

**Celá tabulka**

```json
{
  "steps": [
    {
      "type": "backupTable",
      "table": "AUDIT_LOG",
      "allRows": true
    }
  ]
}
```

**Vybrané skupiny řádků**

```json
{
  "steps": [
    {
      "type": "backupTable",
      "table": "CONFIG_TEST",
      "key": ["CONFIGTYPE"],
      "match": [
        {"CONFIGTYPE": "POA_CONFIG"},
        {"CONFIGTYPE": "BMI"}
      ]
    }
  ]
}
```

### restoreRows — původní hodnoty sloupců

Výběr proběhne při Capture. Restore vrátí každému řádku jeho původní hodnoty podle `key`. Nové řádky ponechá; chybějící přeskočí s upozorněním.

**Vybrané řádky**

```json
{
  "steps": [
    {
      "type": "restoreRows",
      "table": "CONFIG",
      "key": ["CONFIG_KEY"],
      "match": [
        {"CONFIG_KEY": "API_URL"},
        {"CONFIG_KEY": "CALLBACK_URL"}
      ],
      "columns": ["CONFIG_VALUE", "DESCRIPTION"]
    }
  ]
}
```

**Všechny původní řádky**

```json
{
  "steps": [
    {
      "type": "restoreRows",
      "table": "CONFIG",
      "key": ["CONFIG_KEY"],
      "allRows": true,
      "columns": ["CONFIG_VALUE", "DESCRIPTION"]
    }
  ]
}
```

**Složený klíč**

```json
{
  "steps": [
    {
      "type": "restoreRows",
      "table": "TENANT_CONFIG",
      "key": ["TENANT_ID", "CONFIG_KEY"],
      "match": [
        {"TENANT_ID": 10, "CONFIG_KEY": "API_URL"},
        {"TENANT_ID": 20, "CONFIG_KEY": "API_URL"}
      ],
      "columns": ["CONFIG_VALUE", "DESCRIPTION"]
    }
  ]
}
```

### replaceTable — celý obsah

Při Restore nejprve smaže všechny tabulky `replaceTable` v opačném pořadí konfigurace, i když nejsou propojené. Potom vkládá data v pořadí konfigurace. U navázaných tabulek uveď rodiče před potomkem.

**Jedna tabulka**

```json
{
  "steps": [
    {
      "type": "replaceTable",
      "table": "LOOKUP",
      "allRows": true
    }
  ]
}
```

**Navázané tabulky**

```json
{
  "steps": [
    {
      "type": "replaceTable",
      "table": "USERS",
      "allRows": true
    },
    {
      "type": "replaceTable",
      "table": "USERGROUPS",
      "allRows": true
    }
  ]
}
```

### insert — doplnění řádků

Capture uloží celé vybrané řádky. Restore chybějící vloží, shodné existující přeskočí. Stejný klíč s jinými hodnotami znamená konflikt; další řádky zůstanou.

**Vybrané řádky**

```json
{
  "steps": [
    {
      "type": "insert",
      "table": "CONFIG",
      "key": ["CONFIG_KEY"],
      "match": [
        {"CONFIG_KEY": "API_URL"},
        {"CONFIG_KEY": "CALLBACK_URL"}
      ]
    }
  ]
}
```

**Všechny původní řádky**

```json
{
  "steps": [
    {
      "type": "insert",
      "table": "TEST_USERS",
      "key": ["USER_ID"],
      "allRows": true
    }
  ]
}
```

### update — pevné hodnoty

Výběr proběhne po refreshi. Všechny vybrané řádky dostanou stejné hodnoty `set`. Jedna podmínka může vybrat více řádků.

**Konkrétní podmínka**

```json
{
  "steps": [
    {
      "type": "update",
      "table": "CONFIG",
      "key": ["CONFIG_KEY"],
      "match": [
        {"CONFIG_KEY": "API_URL"}
      ],
      "set": {"CONFIG_VALUE": "https://api-test.example.cz", "DESCRIPTION": "Testovací API"}
    }
  ]
}
```

**Více řádků se stejnou podmínkou**

```json
{
  "steps": [
    {
      "type": "update",
      "table": "INTEGRATIONS",
      "key": ["ENVIRONMENT"],
      "match": [
        {"ENVIRONMENT": "PROD"},
        {"ENVIRONMENT": "STAGING"}
      ],
      "set": {"ENABLED": 0}
    }
  ]
}
```

**Všechny řádky**

```json
{
  "steps": [
    {
      "type": "update",
      "table": "NOTIFICATION_SETTINGS",
      "key": ["ID"],
      "allRows": true,
      "set": {"ENABLED": 0, "RECIPIENT_EMAIL": null}
    }
  ]
}
```

### delete — smazání řádků

Maže až po refreshi. Každý běh odstraní všechny aktuálně odpovídající řádky. S `allRows` smaže celý aktuální obsah, včetně řádků přidaných od minulého běhu.

**Vybrané skupiny řádků**

```json
{
  "steps": [
    {
      "type": "delete",
      "table": "TEMP_MESSAGES",
      "key": ["SOURCE", "STATUS"],
      "match": [
        {"SOURCE": "PROD", "STATUS": "PENDING"},
        {"SOURCE": "PROD", "STATUS": "ERROR"}
      ]
    }
  ]
}
```

**Celá tabulka**

```json
{
  "steps": [
    {
      "type": "delete",
      "table": "TEMP_MESSAGES",
      "allRows": true
    }
  ]
}
```

## 5. Spusť Capture a obnovu

**Zastav zápisy aplikací do dotčených tabulek a nech je zastavené až do dokončení validace.** Capture není jeden společný databázový snapshot všech dotazů.

### Přes okno aplikace

Z kořene projektu spusť `DatabaseRefreshUtility.bat`, nebo:

```powershell
powershell.exe -NoProfile -STA -File .\scripts\Start-DatabaseRefreshUtilityUI.ps1
```

1. Před refreshem klikni na **Capture** a počkej na `CAPTURE SUCCESS`.
2. Uchovej celý vytvořený adresář `snapshots/<čas>/`. UI automaticky vybere nový `snapshot.json`.
3. Nech DBA provést refresh.
4. Spusť **Preflight**. Po úspěchu spusť **Restore**.
5. Restore zopakuje Preflight, provede obnovu a následnou validaci. **Validate** lze spustit i samostatně.

Preflight pouze kontroluje databázi; neprovádí zkušební zápisy. Některé chyby, například porušení CHECK constraintu, se proto mohou projevit až při Restore.

### Přes skripty

```powershell
# Před refreshem:
powershell.exe -NoProfile -File .\scripts\Capture.ps1

# Po refreshi od DBA; <cas> nahraď skutečnou složkou:
powershell.exe -NoProfile -File .\scripts\Preflight.ps1 -Snapshot ".\snapshots\<cas>\snapshot.json"
powershell.exe -NoProfile -File .\scripts\Restore.ps1 -Snapshot ".\snapshots\<cas>\snapshot.json"

# Volitelná opakovaná kontrola:
powershell.exe -NoProfile -File .\scripts\Validate.ps1 -Snapshot ".\snapshots\<cas>\snapshot.json"
```

## 6. Kde najdeš výsledky a logy

| Umístění | Obsah |
| --- | --- |
| `snapshots/<čas>/snapshot.json` | Data pro automatickou obnovu. Uchovej celý adresář snapshotu. |
| `snapshots/<čas>/<SCHÉMA>/` | CSV a ruční INSERT SQL pro každou tabulku. |
| `logs/<čas>-<id>-<operace>.log` | Automatický log běhu, přes UI i skripty. |
| `restore-plan.json` vedle snapshotu | Plán opakování; nemaž ho ani neupravuj. |
| `recovery/` vedle snapshotu | Přeskočené řádky `restoreRows` a případný SQL pro jejich ruční doplnění. |

Log běží od načtení konfigurace po výsledek. Obsahuje UTC časy, kroky, počty řádků, volání SQL*Plus, varování, chyby, potvrzení transakcí a konečný `SUCCESS`/`FAILED`. Cesta se vypíše jako `LOG FILE`. Logy se průběžně zapisují, nepřepisují a automaticky nemažou. Při násilném ukončení může chybět závěrečný záznam. Pokud nelze log založit, práce s databází nezačne.

Hesla, hodnoty řádků a celé SQL příkazy se do provozního logu nezapisují. Zálohy a recovery soubory však obsahují databázová data. Chraň je spolu s konfigurací. **Save log** v UI navíc uloží viditelný výpis okna.

`backupTable` a `insert` exportují jen zvolený výběr. Ostatní operace při Capture zálohují celou tabulku. CSV zachovává NULL pomocí sloupců `__IS_NULL`. Snapshot a exporty se před úspěšným dokončením kontrolují pomocí kontrolních součtů.

Ruční SQL exporty se automaticky nespouštějí a neobsahují COMMIT. Před použitím ověř cílová data, strukturu a vazby; běžný export předpokládá, že v cíli vkládané řádky nejsou. Recovery SQL vkládá jen stále chybějící klíče. Pokud má tabulka identity sloupce, může být nutná ruční příprava INSERT. Chyba tvorby recovery souboru nevrací již potvrzenou obnovu.

## 7. Co je důležité při obnově

- Každé schéma má vlastní transakci a COMMIT. Chyba vrátí změny aktuálního schématu, ale dříve dokončená schémata zůstávají potvrzená. Chyba následné samostatné validace už COMMIT nevrátí.
- Tabulky se během obnovy zamykají. Obsazený zámek způsobí chybu; TRUNCATE se nepoužívá.
- Po opravě příčiny lze Restore zopakovat se stejným snapshotem a plánem. Při ztrátě spojení během COMMIT ověř stav validací. Nespouštěj více obnov stejného snapshotu současně.
- UPDATE ukládá vybrané hodnoty `key` a počet řádků každé skupiny. Opakování vyžaduje stejné počty a ověřuje všechny nastavené hodnoty. Výměnu řádků za jiné se stejným klíčem a počtem nepozná. Samostatná validace UPDATE potřebuje uložený plán.
- `backupTable`, `restoreRows`, `replaceTable` a `insert` musí být jediným krokem pro danou tabulku. Více UPDATE/DELETE je možné, ale UPDATE musí mít stejný seznam `key` a nesmějí se překrývat. DELETE nesmí zasáhnout řádky potřebné pro validaci UPDATE.
- Aktivní triggery musí vyřešit DBA. Nástroj nevypíná triggery ani constraints. U `replaceTable` musí být závislé děti také nahrazované a uvedené za rodičem; cykly a samoodkazy se odmítají.
- Prostředí nesmí mít příchozí cizí klíče mezi schématy. Účet nemusí všechny takové vazby vidět. Kontroly používají `ALL_CONSTRAINTS` a `USER_CONSTRAINTS`, nevyžadují `SYS.DBA_CONSTRAINTS`.
- Připojení určuje cíl. Nástroj ověřuje účet, ale nezávisle neověřuje identitu databáze. Změnu cíle skrytou pod stejným TNS aliasem nerozpozná.

## 8. Datové typy a řešení chyb

Podporované typy: `CHAR`, `VARCHAR2`, `NCHAR`, `NVARCHAR2`, `NUMBER`, `DATE`, `TIMESTAMP`, `TIMESTAMP WITH TIME ZONE`. Například `CLOB`, `BLOB`, `RAW`, `TIMESTAMP WITH LOCAL TIME ZONE` a příliš velké exportované řádky nejsou podporované. I sloupec mimo `columns` může Capture zablokovat, protože záloha obsahuje celé řádky.

`insert` a `replaceTable` nepodporují identity ani neviditelné sloupce. Virtuální sloupce se nevkládají. UPDATE nesmí měnit identity ani virtuální sloupce.

Datum zapisuj jako `2026-09-25 14:30:00`, timestamp jako `2026-09-25 14:30:00.123456789`, časovou zónu přidej jako ` +02:00`. Zachovává se offset, ne název regionu. Velká přesná čísla v JSON zapisuj jako řetězce, aby se neztratila přesnost. Konfigurace a exporty používají UTF-8.

| Problém | Co ověřit |
| --- | --- |
| `ORA-12154` | TNS alias musí fungovat v klientovi z `sqlplusPath`; zkontroluj `tnsnames.ora` a `TNS_ADMIN`. |
| `selected keys must be unique and non-null` | U `restoreRows`/`insert` zkontroluj duplicity a NULL ve vybraných klíčích. S `allRows` v celé tabulce. |
| `UPDATE group count changed` | Od vytvoření plánu se změnil počet řádků některé skupiny. Prověř data a souběžné zápisy; plán ručně neupravuj. |
| `layout changed` | Struktura tabulky po refreshi nesouhlasí se snapshotem. |
| Chybí řádek při `restoreRows` | Obnova jej přeskočí; podrobnosti najdeš v `recovery/`. |

## 9. Starší konfigurace a testy

Aktuální snapshot má **verzi 5**. Verze 1–4 se odmítají. Při přechodu vytvoř nový Capture **před refreshem**. Pokud refresh již proběhl, starou zálohu dokonči původní verzí nástroje; snapshot nepřeváděj ručně. Nový refresh vždy vyžaduje nový Capture. Změna připojení nebo kroků také vyžaduje nový Capture; změna hesla ne.

Staré `keyValues` nahraď polem objektů `match`. Odstraň `maxRows`, `backupMaxRows`, `maxInsertRows`, `maxDeleteRows` a `expectedRows`. Pro celou tabulku přidej `allRows: true`. Starý `credentials.json` se nečte: účty patří do `database.json`, `tnsAlias` se jmenuje `host`. `expectedTarget`, `dbUniqueName` a `conName` se nenastavují.

Testy a požadavky pro testovací Oracle schéma popisuje [tests/README.md](tests/README.md).

```powershell
powershell.exe -NoProfile -File .\tests\Test-DatabaseRefreshUtility.ps1
powershell.exe -NoProfile -STA -File .\tests\Test-DatabaseRefreshUtilityUI.ps1
```
