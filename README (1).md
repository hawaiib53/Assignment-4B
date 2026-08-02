#!/usr/bin/env bash
# Convert every meeting-notes and market-info source file into plain markdown
# so an agent (or a human) can read the content without opening Word.
#
# Reads only from inputs/meeting-notes and inputs/market-info under the repo
# root — never fetches anything over the network.
#
# Usage: extract_text.sh <work_dir>
set -euo pipefail

REPO_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/../../../.." && pwd)"
WORK_DIR="${1:?Usage: extract_text.sh <work_dir>}"
mkdir -p "$WORK_DIR"

extract_dir() {
  local category="$1" dir="$2" f base name ext out
  [[ -d "$dir" ]] || return 0
  for f in "$dir"/*; do
    [[ -f "$f" ]] || continue
    base="$(basename "$f")"
    [[ "$base" == "README.md" ]] && continue
    name="${base%.*}"
    ext="${base##*.}"
    out="$WORK_DIR/${category}__${name}.md"
    case "${ext,,}" in
      doc)
        soffice --headless --convert-to docx --outdir "$WORK_DIR" "$f" >/dev/null 2>&1
        pandoc -t markdown "$WORK_DIR/${name}.docx" -o "$out"
        rm -f "$WORK_DIR/${name}.docx"
        ;;
      docx)
        pandoc -t markdown "$f" -o "$out"
        ;;
      pdf)
        pandoc -t markdown "$f" -o "$out" 2>/dev/null || pdftotext "$f" "$out"
        ;;
      txt|md)
        cp "$f" "$out"
        ;;
      *)
        echo "Skipping unsupported file: $base ($category)" >&2
        ;;
    esac
  done
}

extract_dir meeting-notes "$REPO_ROOT/inputs/meeting-notes"
extract_dir market-info "$REPO_ROOT/inputs/market-info"

echo "Extracted text written to $WORK_DIR:"
ls -1 "$WORK_DIR" 2>/dev/null || echo "(nothing extracted — both input folders are empty)"
