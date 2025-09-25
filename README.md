# Public firmware distribution – specifikace a postup
*Aktualizováno: 2025-09-25*

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

## 5) Workflow pro GitHub Actions (release beta)
- Tento script automaticky releasuje betu jako drawf s postfixem souborů -beta. Není veřejně vidět.
- Spouští se automaticky pushem do main. Je nutné mít souboury pojmenovane dle sep SemVer (v1.2.3)
- Pokud se script nespustí sám je možné action spustit ručně, ale je potřeba vyplnit verzi (1.2.3)

Vložte do **`.github/workflows/release_beta.yml`**:

```yaml
name: Upload BETA firmware (multi-variant)

on:
  push:
    branches: [ main ]             # změň na 'main', pokud používáš main
    paths:
      - 'release-assets/**'
  workflow_dispatch:
    inputs:
      version:
        description: 'SemVer verze (např. 1.2.3) — při ručním spuštění'
        required: true

jobs:
  beta:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    env:
      GH_TOKEN: ${{ github.token }}

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      # 1) Zjisti verzi z názvů bez -beta (…-vX.Y.Z.bin) nebo z inputu
      - name: Detect version (from filenames without -beta)
        id: detect
        shell: bash
        run: |
          set -e
          if [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
            VERSION="${{ github.event.inputs.version }}"
          else
            test -d release-assets || { echo "::error::Missing release-assets/"; exit 1; }
            shopt -s nullglob
            FILES=(release-assets/*.bin)
            (( ${#FILES[@]} > 0 )) || { echo "::error::No .bin files in release-assets/"; exit 1; }
            FIRST="$(basename "${FILES[0]}")"
            if [[ "$FIRST" =~ -v([0-9]+\.[0-9]+\.[0-9]+)\.bin$ ]]; then
              VERSION="${BASH_REMATCH[1]}"
            else
              echo "::error::Cannot parse version from filename: $FIRST"
              exit 1
            fi
            # ověř, že všechny mají stejnou verzi (bez -beta)
            for f in "${FILES[@]}"; do
              BN="$(basename "$f")"
              [[ "$BN" =~ -v${VERSION}\.bin$ ]] || { echo "::error::Mixed versions detected: $BN"; exit 1; }
            done
          fi
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"
          echo "Detected version: $VERSION"

      # 2) Zjisti MODEL prefix (např. EDCDEV) a validuj požadované mutace (bez -beta v názvu)
      - name: Validate required variants & detect MODEL
        id: model
        shell: bash
        run: |
          set -e
          VERSION='${{ steps.detect.outputs.version }}'
          test -d release-assets || { echo "::error::Missing release-assets/"; exit 1; }
          ls -la release-assets

          # vyzvedni MODEL z prvního souboru (vše před '-P..B..-v')
          FIRST="$(basename "$(ls release-assets/*.bin | head -n1)")"
          if [[ "$FIRST" =~ ^([A-Z0-9]+)-P(6|8|10|15)B(1|2)-v${VERSION}\.bin$ ]]; then
            MODEL="${BASH_REMATCH[1]}"
          else
            echo "::error::Filename format invalid (expected <MODEL>-P..B..-v${VERSION}.bin): $FIRST"
            exit 1
          fi

          # uprav, pokud nevydáváš všechny mutace
          REQUIRED="P15B1 P10B1 P8B1 P6B1 P15B2 P8B2 P6B2"

          FAIL=0
          for MUT in $REQUIRED; do
            FILE="${MODEL}-${MUT}-v${VERSION}.bin"
            if [ ! -f "release-assets/$FILE" ]; then
              echo "::error::Missing expected asset: $FILE"
              FAIL=1
            fi
            echo "$FILE" | grep -Eq "^${MODEL}-P(6|8|10|15)B(1|2)-v${VERSION}\.bin$" || { echo "::error::Filename invalid: $FILE"; FAIL=1; }
          done
          [ $FAIL -eq 0 ] || exit 1

          echo "model=$MODEL" >> "$GITHUB_OUTPUT"
          echo "Detected MODEL: $MODEL"

      # 3) Připrav výstupní složku 'out/' s přejmenovanými -beta soubory a jejich SHA-256
      - name: Prepare -beta artifacts (copy & rename)
        shell: bash
        run: |
          set -e
          VERSION='${{ steps.detect.outputs.version }}'
          MODEL='${{ steps.model.outputs.model }}'
          mkdir -p out
          shopt -s nullglob
          for f in release-assets/*.bin; do
            BN="$(basename "$f")"
            # převeď <MODEL>-P..B..-vX.Y.Z.bin -> <MODEL>-P..B..-vX.Y.Z-beta.bin
            NEW="${BN%.bin}-beta.bin"
            cp "release-assets/$BN" "out/$NEW"
          done
          cd out
          for f in *-beta.bin; do
            sha256sum "$f" | awk '{print $1 "  " FILENAME}' FILENAME="$f" > "$f.sha256"
          done
          ls -la

      # 4) Pokud existuje draft pro tuto verzi, smaž shodné -beta assety (aby šel re-run)
      - name: Clean existing -beta assets in existing draft (if any)
        shell: bash
        run: |
          set -e
          VERSION='${{ steps.detect.outputs.version }}'
          gh api repos/${GITHUB_REPOSITORY}/releases?per_page=100 > list.json
          RID=$(jq -r --arg ver "v${VERSION}" '
            .[] | select(.draft==true) |
            select(
              (.name == ("Firmware " + $ver + " BETA")) or
              (.tag_name == ($ver + "-beta"))
            ) | .id
          ' list.json | head -n1)
          if [ -n "$RID" ] && [ "$RID" != "null" ]; then
            echo "Found existing draft release id=$RID — cleaning *-beta.* assets"
            gh api repos/${GITHUB_REPOSITORY}/releases/$RID > rel.json
            for f in out/*; do
              BASENAME=$(basename "$f")
              AID=$(jq -r --arg n "$BASENAME" '.assets[] | select(.name==$n) | .id' rel.json)
              if [ -n "$AID" ] && [ "$AID" != "null" ]; then
                echo "Deleting existing asset $BASENAME (id=$AID)"
                gh api -X DELETE repos/${GITHUB_REPOSITORY}/releases/assets/$AID >/dev/null
              fi
            done
          else
            echo "No existing draft found — a new one will be created."
          fi

      # 5) Vytvoř/aktualizuj BETA release jako DRAFT a nahraj *-beta.* artefakty
      - name: Create or Update BETA release (as draft)
        uses: softprops/action-gh-release@v2
        with:
          tag_name: v${{ steps.detect.outputs.version }}-beta
          name: Firmware v${{ steps.detect.outputs.version }} BETA
          draft: true
          prerelease: false
          files: |
            out/*-beta.bin
            out/*-beta.bin.sha256
```

---

## 7) Workflow pro GitHub Actions (release beta)
- Script se nespouští sám. Je porřeba action spustit ručně,je potřeba vyplnit verzi (1.2.3) kterou povyšuji 

Vložte do **`.github/workflows/promote_to_prerelease.yml`**:
```yaml
name: Promote BETA to Pre-release (rename assets)
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Základní verze (např. 1.3.0)'
        required: true
      rc_suffix:
        description: 'Suffix pro soubory (např. -rc1), prázdné = nepřejmenovávat'
        required: false
        default: '-rc1'

jobs:
  promote:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    env:
      GH_TOKEN: ${{ github.token }}
      VERSION: ${{ github.event.inputs.version }}
      RC_SUFFIX: ${{ github.event.inputs.rc_suffix }}

    steps:
      - name: Install jq
        run: sudo apt-get update && sudo apt-get install -y jq

      - name: Find DRAFT beta release (no tag yet)
        id: find
        run: |
          set -e
          # Stáhni posledních 100 releasů (drafty i publikované)
          gh api repos/${GITHUB_REPOSITORY}/releases?per_page=100 > list.json

          # Hledej DRAFT, který odpovídá dané verzi:
          # 1) name == "Firmware vX.Y.Z BETA"
          # 2) nebo tag_name == "vX.Y.Z-beta" (někdy ho draft má vyplněný, ale tag v gitu ještě neexistuje)
          RID=$(jq -r --arg ver "v${VERSION}" '
            .[] | select(.draft==true) |
            select(
              (.name == ("Firmware " + $ver + " BETA")) or
              (.tag_name == ($ver + "-beta"))
            ) | .id
          ' list.json | head -n1)

          if [ -z "$RID" ] || [ "$RID" = "null" ]; then
            echo "::error::Draft release for version v${VERSION} not found. Check the name/tag or that the BETA workflow finished."
            exit 1
          fi

          echo "rid=$RID" >> $GITHUB_OUTPUT
          # Ulož i celý JSON releasu pro pozdější práci s assety
          gh api repos/${GITHUB_REPOSITORY}/releases/$RID > rel.json
          echo "Found DRAFT release id=$RID"

      - name: Publish as PRE-RELEASE (prerelease=true, draft=false)
        run: |
          gh api -X PATCH repos/${GITHUB_REPOSITORY}/releases/${{ steps.find.outputs.rid }} \
            -f draft=false -f prerelease=true \
            -f name="Firmware v${VERSION}${RC_SUFFIX:+ ${RC_SUFFIX#-} } (pre-release)"

      - name: Optionally rename assets (add suffix like -rc1)
        if: ${{ env.RC_SUFFIX != '' }}
        run: |
          set -e
          # Po publishi si případně načti čerstvou verzi releasu (asset ID zůstávají)
          ASSETS=$(jq -c '.assets[] | {id: .id, name: .name}' rel.json)
          while read -r A; do
            AID=$(echo "$A" | jq -r '.id')
            ANAME=$(echo "$A" | jq -r '.name')
            case "$ANAME" in
              *.bin|*.sha256)
                BASE="${ANAME%.*}"
                EXT="${ANAME##*.}"
                # Cíl: ...-vX.Y.Z[-rcN].ext → přidat -rcN pokud chybí
                if [[ "$BASE" == *"-v${VERSION}" ]]; then
                  NEW="${BASE}${RC_SUFFIX}.${EXT}"
                  echo "Renaming $ANAME -> $NEW"
                  gh api -X PATCH repos/${GITHUB_REPOSITORY}/releases/assets/${AID} -f name="$NEW" >/dev/null
                fi
                ;;
            esac
          done <<< "$ASSETS"
```

---

## 8) Workflow pro GitHub Actions (promote to stable)
- Script se nespouští sám. Je porřeba action spustit ručně,je potřeba vyplnit verzi (1.2.3) kterou povyšuji 

Vložte do **`.github/workflows/promote_to_stable.yml`**:
```yaml
name: Promote Pre-release to Stable (Latest)

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Finální verze (např. 1.3.0)'
        required: true

jobs:
  stable:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    env:
      GH_TOKEN: ${{ github.token }}
      VERSION: ${{ github.event.inputs.version }}

    steps:
      - name: Install jq and curl
        run: sudo apt-get update && sudo apt-get install -y jq curl

      - name: Get pre-release (tag vX.Y.Z-beta)
        id: getpre
        run: |
          set -e
          RJSON=$(gh api repos/${GITHUB_REPOSITORY}/releases/tags/v${VERSION}-beta || true)
          if [ -z "$RJSON" ] || [ "$RJSON" = "null" ]; then
            echo "::error::Pre-release (v${VERSION}-beta) not found."
            exit 1
          fi
          echo "$RJSON" > pre.json

      - name: Download pre-release assets
        run: |
          mkdir in && cd in
          ASSETS_URLS=$(jq -r '.assets[].browser_download_url' ../pre.json)
          for URL in $ASSETS_URLS; do
            echo "downloading $URL"
            curl -L -O "$URL"
          done
          ls -la

      - name: Rename assets to final names
        run: |
          cd in
          for f in *; do
            # Odstraň '-rcX' / '-beta' ze jména verze a přepiš na -v${VERSION}
            NEW=$(echo "$f" | sed -E 's/-v([0-9]+\.[0-9]+\.[0-9]+)-(rc[0-9]+|beta(\.[0-9]+)?)\./-v\1./g')
            NEW=$(echo "$NEW" | sed -E "s/-v[0-9]+\.[0-9]+\.[0-9]+/-v${VERSION}/g")
            if [ "$f" != "$NEW" ]; then
              echo "$f -> $NEW"
              mv "$f" "$NEW"
            fi
          done
          ls -la

      - name: Create STABLE release and upload
        uses: softprops/action-gh-release@v2
        with:
          tag_name: v${{ env.VERSION }}
          name: Firmware v${{ env.VERSION }}
          draft: false
          prerelease: false
          files: |
            in/*

```

---

## 9) Šablona release notes (volitelné)
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

## 10) Bezpečnostní doporučení
- Publikovat **SHA-256** ke každému souboru.
- Zvážit **digitální podpis** (Ed25519/ECDSA) a zveřejnit **veřejný klíč**.
- Do release notes uvádět **min. verzi bootloaderu** a kompatibilitu s app/cloud.

---

## 11) FAQ
**Proč mít P/B v názvu?**  
Je to čitelné pro lidi i stroje – `P15` = 15 J, `B2` = VOSS.

**Mohu vydat vše jedním releasem?**  
Ano. Tag `vX.Y.Z` → jeden release → nahrát všechny mutace (P×B) jako assets.

**Jak odkazovat na „poslední stabilní“?**  
Přes URL `/releases/latest/download/<soubor>` – vždy míří na poslední stabilní (ne pre-release).
