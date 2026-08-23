# XSS Candidate Scanner

Jednoduchý Python nástroj pro vyhledávání potenciálních **XSS (Cross-Site Scripting)** problémů ve webových aplikacích. Scanner prochází stránky v rámci stejného originu, analyzuje JavaScript, formuláře a URL parametry a vytváří report nalezených kandidátů.

> **Upozornění:** Používejte pouze na webových aplikacích, které vlastníte nebo k jejichž testování máte výslovné oprávnění.

## Funkce

Scanner umí:

* procházet stránky v rámci stejného originu,
* hledat potenciální DOM XSS zdroje a sinky,
* analyzovat inline JavaScript,
* analyzovat externí JavaScript ze stejného originu,
* detekovat formuláře a jejich vstupní pole,
* kontrolovat reflexi GET parametrů v HTTP odpovědi,
* odstranit fragmenty (`#...`) z nalezených URL,
* omezit maximální počet procházených stránek,
* uložit výsledky do textového reportu.

Scanner hledá **kandidáty na zranitelnosti**. Nález tedy automaticky neznamená, že je aplikace skutečně zranitelná.

## Požadavky

* Python 3
* `requests`
* `beautifulsoup4`

## Instalace

Naklonujte nebo stáhněte projekt a nainstalujte závislosti:

```bash
pip install requests beautifulsoup4
```

Doporučeno je použít virtuální prostředí:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install requests beautifulsoup4
```

Ve Windows lze prostředí aktivovat například:

```powershell
.venv\Scripts\Activate.ps1
```

## Použití

Základní spuštění:

```bash
python3 xss_scanner.py https://example.com
```

Scanner začne na zadané URL a bude následovat nalezené odkazy v rámci stejného originu.

### Dostupné argumenty

```text
url             Výchozí URL pro scan

--max-pages     Maximální počet procházených stránek
                Výchozí hodnota: 50

--timeout       Timeout HTTP požadavku v sekundách
                Výchozí hodnota: 10

--output        Soubor pro výsledný report
                Výchozí hodnota: xss_report.txt
```

Například:

```bash
python3 xss_scanner.py https://example.com \
    --max-pages 100 \
    --timeout 15 \
    --output report.txt
```

## Jak scanner funguje

### 1. Crawling

Scanner začne na zadané URL a pomocí fronty postupně prochází nalezené odkazy.

Z HTML získává URL z atributů:

```text
href
src
action
```

Následuje pouze HTTP/HTTPS URL se stejným originem jako výchozí stránka.

### 2. DOM XSS kandidáti

JavaScript je kontrolován na současný výskyt potenciálních **sources** a **sinks**.

Mezi sledované zdroje patří například:

```javascript
location.hash
location.search
location.href
document.URL
document.documentURI
document.referrer
window.name
localStorage
sessionStorage
postMessage
```

Mezi sledované sinky patří například:

```javascript
element.innerHTML = ...
element.outerHTML = ...
insertAdjacentHTML(...)
document.write(...)
document.writeln(...)
eval(...)
setTimeout(...)
setInterval(...)
Function(...)
```

Pokud scanner ve stejném analyzovaném JavaScriptu najde source i sink, označí ho jako:

```text
DOM XSS candidate
```

Tato kontrola neprovádí skutečnou analýzu toku dat. Samotná přítomnost source a sinku proto není důkazem DOM XSS.

### 3. Reflected XSS kandidáti

Pokud URL obsahuje query parametry, například:

```text
https://example.com/search?q=test
```

scanner postupně nahradí hodnotu každého parametru unikátním markerem:

```text
XSSCANDIDATE82461
```

Poté odešle HTTP požadavek a zkontroluje, zda se marker objevil v odpovědi.

Pokud ano, vytvoří nález:

```text
Reflected XSS candidate
```

Reflexe hodnoty sama o sobě neznamená XSS. Hodnota může být například bezpečně HTML escapována.

### 4. Formuláře

Scanner zaznamenává HTML formuláře obsahující pojmenovaná vstupní pole.

Například:

```html
<form method="GET" action="/search">
    <input name="q">
</form>
```

může vytvořit položku:

```text
Input surface
GET form fields: q
```

Tato položka pouze identifikuje potenciální vstupní plochu aplikace.

## Výstup

Po dokončení scanu se vytvoří textový report.

Příklad:

```text
XSS CANDIDATE REPORT
================================================================================

Pages scanned: 12
Candidates: 3

[1] Reflected XSS candidate
URL: https://example.com/search?q=XSSCANDIDATE82461
Evidence: Parameter 'q' reflected into response
--------------------------------------------------------------------------------
```

## Omezení

Scanner je záměrně jednoduchý a jeho výsledky vyžadují manuální ověření.

Aktuálně například:

* neprovádí plnohodnotnou JavaScript data-flow analýzu,
* nerozlišuje automaticky bezpečně escapovanou reflexi od nebezpečné,
* nespouští JavaScript v prohlížeči,
* neanalyzuje DOM vytvořený až za běhu aplikace,
* nepřihlašuje se automaticky do aplikace,
* netestuje POST formuláře,
* neřeší komplexní SPA routing,
* nepovažuje nalezeného kandidáta automaticky za potvrzenou XSS zranitelnost.

Proto je vhodné výsledky chápat jako seznam míst určených k další bezpečnostní analýze.

## Struktura reportovaných nálezů

Scanner používá tři základní typy:

| Typ                       | Význam                                              |
| ------------------------- | --------------------------------------------------- |
| `DOM XSS candidate`       | JavaScript obsahuje sledovaný source i sink         |
| `Reflected XSS candidate` | Hodnota URL parametru se objevila v HTTP odpovědi   |
| `Input surface`           | Byl nalezen formulář s pojmenovanými vstupními poli |

## Bezpečné použití

Scanner generuje HTTP požadavky proti cílové aplikaci a crawling může navštívit větší množství endpointů.

Před použitím:

1. Získejte oprávnění k testování cílového systému.
2. Nastavte rozumnou hodnotu `--max-pages`.
3. Nepouštějte scanner proti systémům, kde crawling může vyvolat nežádoucí operace.
4. Nálezy před reportováním manuálně ověřte.

##
