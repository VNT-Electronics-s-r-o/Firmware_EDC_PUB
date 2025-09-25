# Public firmware distribution – specifikace a postup
*Aktualizováno: 2025-09-23*

Tento dokument popisuje **značení výkonu a brandu** v názvu buildů, pravidla verzování a **postup vydávání** (stabilní i pre-release) pro veřejnou distribuci firmware na GitHubu.

---

## 1) Značení výkonu a brandu v názvu
Používáme formát **EDCXXX-P<výkon>B<brand>-v<major>.<minor>.<patch>**

```
EDCXXX-P<výkon>B<brand>-v<major>.<minor>.<patch>
```

### Význam částí
- **EDCXXX** – mateřská platforma (EDC).
- **P<výkon>** – **skutečný výkon v joulech**, např. `P15`, `P10`, `P8`, `P6`.
- **B<brand>** – kód brandu (značky):
  - `B1` = fencee
  - `B2` = VOSS
  - `B3+` = rezerva (pro budoucí brandy)
- **vX.Y.Z** – verze firmware dle [SemVer](https://semver.org/) (MAJOR.MINOR.PATCH).

### Příklady
- `EDCXXX-P15B1-v1.1.0` → **EDC150**, 15 J, **fencee**, verze 1.1.0  
- `EDCXXX-P15B2-v1.1.0` → **SMD20**, 15 J, **VOSS**, verze 1.1.0  
- `EDCXXX-P8B1-v2.0.0`  → **EDC80**, 8 J, **fencee**, verze 2.0.0  
- `EDCXXX-P6B2-v1.5.2`  → **SMD8**, 6 J, **VOSS**, verze 1.5.2  

### Mapování výkon ↔ model (brand 1/2)
| P-kód | Výkon | Brand 1 (fencee) | Brand 2 (VOSS) |
|:----:|:-----:|:------------------|:---------------|
| P15  | 15 J  | EDC150            | SMD20          |
| P10  | 10 J  | EDC100            | –              |
| P8   | 8 J   | EDC80             | SMD11          |
| P6   | 6 J   | EDC60             | SMD8           |

> Pozn.: Jedna binárka může mít **aliasy** (např. `EDC150-v1.1.0.bin` a `SMD20-v1.1.0.bin`), pokud je obsah identický.

---

## 2) Pravidla verzování (SemVer)
- **MAJOR** – nekompatibilní změny (bootloader, formát konfigurace, protokol).
- **MINOR** – nové funkce zpětně kompatibilní.
- **PATCH** – bugfixy a drobné úpravy bez dopadu na kompatibilitu.

### Před-release (RC/beta)
- Před-release tagy: `v1.2.0-rc1`, `v1.2.0-beta.2`.
- Finální stabilní: `v1.2.0` (bez suffixu).
- GitHub označí **nejnovější stabilní** jako **Latest**; před-release se do „Latest“ nepočítají.

---

## 3) Strategie releasů
- **Jeden tag = jedna verze SW** (např. `v1.1.0`).  
- Release obsahuje **všechny mutace** (P×B) jako **assets**:
  ```
  EDCXXX-P15B1-v1.1.0.bin
  EDCXXX-P10B1-v1.1.0.bin
  EDCXXX-P8B1-v1.1.0.bin
  EDCXXX-P6B1-v1.1.0.bin
  EDCXXX-P15B2-v1.1.0.bin
  EDCXXX-P8B2-v1.1.0.bin
  EDCXXX-P6B2-v1.1.0.bin
  ```
- Ke každé binárce publikujeme **`.sha256`** (a volitelně **`.sig`** + `public.key`).

---

## 4) Doporučené názvy souborů (assets)
- Primární (technické): `EDCXXX-P15B1-v1.1.0.bin`
- Alias (tržní název, volitelné): `EDC150-v1.1.0.bin`, `SMD20-v1.1.0.bin`

> Alias usnadní koncovým uživatelům orientaci; checksum bude stejný jako u primární binárky.

---

## 5) Workflow pro GitHub Actions (pre-release + latest)
Vložte do **`.github/workflows/release.yml`**:

```yaml
name: Publish multi-variant firmware release

on:
  push:
    tags:
      - 'v*'      # spustí se pro v1.2.0 i v1.2.0-rc1

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Parse tag (detect prerelease)
        id: tag
        run: |
          TAG="${{ github.ref_name }}"
          echo "tag=$TAG" >> $GITHUB_OUTPUT
          if [[ "$TAG" == *"-"* ]]; then
            echo "prerelease=true" >> $GITHUB_OUTPUT
            echo "name_suffix= (pre-release)" >> $GITHUB_OUTPUT
          else
            echo "prerelease=false" >> $GITHUB_OUTPUT
            echo "name_suffix=" >> $GITHUB_OUTPUT
          fi

      # Očekáváme, že v pracovním adresáři existuje složka release-assets/ se soubory
      - name: Ensure assets exist
        run: |
          test -d release-assets || { echo "Missing release-assets/"; exit 1; }
          ls -la release-assets

      - name: Generate SHA-256 (if missing)
        run: |
          cd release-assets
          shopt -s nullglob
          for f in *.bin; do
            [ -f "$f.sha256" ] || sha256sum "$f" | awk '{{print $1 "  " FILENAME}}' FILENAME="$f" > "$f.sha256"
          done

      - name: Create GitHub Release + upload assets
        uses: softprops/action-gh-release@v2
        with:
          tag_name: ${{ steps.tag.outputs.tag }}
          name: Firmware ${{ steps.tag.outputs.tag }}${{ steps.tag.outputs.name_suffix }}
          draft: false
          prerelease: ${{ steps.tag.outputs.prerelease }}
          files: |
            release-assets/*
```

### Jak workflow používat (rychlý návod)
1. Připrav složku **`release-assets/`** se všemi `.bin` pro danou verzi (a případnými aliasy).  
2. (Volitelné) Přidej vlastní `.sha256` a `.sig`; jinak se SHA-256 vygenerují automaticky.  
3. Vytvoř **tag**:
   - Pre-release: `git tag v1.2.0-rc1 && git push origin v1.2.0-rc1`
   - Stabilní: `git tag v1.2.0 && git push origin v1.2.0`
4. Po doběhnutí Actions vznikne **Release** s nahranými assets.  
   - Stabilní release GitHub označí jako **Latest**.

---

## 6) Šablona release notes (volitelné)
```md
## Overview
Tag: v1.1.0
Models covered:
- P15B1 (EDC150), P10B1 (EDC100), P8B1 (EDC80), P6B1 (EDC60)
- P15B2 (SMD20),  P8B2 (SMD11),  P6B2 (SMD8)

## Changes
- Added: ...
- Improved: ...
- Fixed: ...

## Compatibility
- Bootloader: BL ≥ 1.2
- App: Android ≥ 1.8.0, iOS ≥ 1.8.0
- Cloud API: ≥ 3.1

## Verification
- SHA-256 checksums attached
- (optional) Signatures *.sig and public.key
```

---

## 7) Bezpečnostní doporučení
- Publikovat **SHA-256** ke každému souboru.
- Zvážit **digitální podpis** (Ed25519/ECDSA) a zveřejnit **veřejný klíč**.
- Do release notes uvádět **min. verzi bootloaderu** a kompatibilitu s app/cloud.

---

## 8) FAQ
**Proč mít P/B v názvu?**  
Je to čitelné pro lidi i stroje – `P15` = 15 J, `B2` = VOSS.

**Mohu vydat vše jedním releasem?**  
Ano. Tag `vX.Y.Z` → jeden release → nahrát všechny mutace (P×B) jako assets.

**Jak odkazovat na „poslední stabilní“?**  
Přes URL `/releases/latest/download/<soubor>` – vždy míří na poslední stabilní (ne pre-release).
