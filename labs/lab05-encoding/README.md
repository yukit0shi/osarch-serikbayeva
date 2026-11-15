# Lab 5 — Encoding forensics

Name: <Твоя Фамилия> Aisulu · Date: 16.11.2026 · Tools: Git Bash (file, xxd, iconv), Excel, LibreOffice Calc

## What I received
- **Excel:** Standard Russian letters opened fine, while Kazakh-specific characters broke into wrong symbols (for example, "Қ" appeared as "Ќ" and "Ә" as "Ј").
- **LibreOffice Calc:** The import wizard defaulted to `Cyrillic (Windows-1251)` with `;` as the separator, showing the exact same corrupted Kazakh letters as Excel.
- **Terminal (`cat`):** Cyrillic text turned into unreadable replacement blocks because the shell expects UTF-8 rather than single-byte encodings.

## Diagnosis (made before the fix)
- **`file` output:** `Non-ISO extended-ASCII text, with CRLF line terminators` (`evidence/file-before.txt`).
- **First 16 bytes:** `49 44 3b d4 e0 ec e8 eb e8 ff 3b c8 ec ff 3b ce` (`evidence/hexdump-before.txt`).
- **Encoding:** `KZ-1048` (named `RK1048` in `iconv`) — each Cyrillic character occupies a single byte (e.g., `"А"` in `"Аружан"` is `c0`, `"И"` in `"Имя"` is `c8`, `"Г"` in `"Город"` is `c3`), and Kazakh letters use KZ-1048 code points (such as `a3` for `"Ә"` and `8d` for `"Қ"`), which standard `Windows-1251` decodes incorrectly.
- **Line ending:** `CRLF` — the header row ends with the two-byte sequence `0d 0a`.
- **BOM:** No BOM — the file begins immediately with ASCII `49 44` (`"ID"`), with no `EF BB BF` signature.
- **Separator:** Semicolon `;` (`3b`).

## Fix decisions & script
I converted the encoding to **UTF-8 with BOM** (`EF BB BF`) so that Windows Excel detects UTF-8 automatically when opened via double-click, and preserved **CRLF** line endings for consistent compatibility with Windows spreadsheet applications.

```sh
#!/bin/sh
in=evidence/staff-broken.csv
out=evidence/staff-fixed.csv
iconv -f RK1048 -t UTF-8 "$in" > "$out"
printf '\357\273\277' | cat - "$out" > tmp && mv tmp "$out"
```
Full script: `scripts/fix.sh`

## Verification
- `evidence/file-after.txt` verifies that `staff-fixed.csv` is `UTF-8 (with BOM)` with `CRLF` line terminators, and `xxd` confirms two-byte `d0 xx` / `d1 xx` sequences for Cyrillic letters.
- `evidence/opens-correctly.png` shows all three applications side by side with Russian and Kazakh names rendering accurately (in Git Bash, a few Kazakh glyphs appear slightly spaced due to the default console font, while the underlying UTF-8 bytes and both spreadsheet views are completely intact).

## What did not work the first time
On my first attempt, I ran `iconv -f RK1048 -t UTF-8` without prepending a UTF-8 BOM. Although `cat` in the terminal displayed every name properly, double-clicking the output file in Windows Excel still produced mojibake (`Р¤Р°РјРёР»РёСЏ`) because Excel falls back to a legacy code page unless the `EF BB BF` signature is present at the start of the file. Prepending the BOM bytes in `scripts/fix.sh` resolved the issue.

## Evidence
- `evidence/staff-broken.csv` — original broken file (untouched)
- `evidence/staff-fixed.csv` — converted UTF-8 with BOM file
- `evidence/file-before.txt` — `file` command output prior to conversion
- `evidence/hexdump-before.txt` — initial hex dump proving single-byte encoding, CRLF, and no BOM
- `evidence/file-after.txt` — `file` and `xxd` verification after conversion
- `evidence/opens-correctly.png` — verification screenshot across Excel, LibreOffice Calc, and Git Bash
