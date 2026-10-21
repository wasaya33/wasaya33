name: Update Profile Stats

on:
  schedule:
    - cron: "0 */3 * * *"   # every 3 hours
  workflow_dispatch:          # lets you run it manually from the Actions tab
  push:
    branches: [main]
    paths:
      - ".github/workflows/update-stats.yml"

permissions:
  contents: write

concurrency:
  group: update-stats
  cancel-in-progress: true

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Fetch stat cards
        run: |
          mkdir -p assets
          USER_NAME="wasaya33"

          fetch() {
            local url="$1" out="$2" tmp="assets/$2.tmp"
            if curl -fsSL --retry 3 --retry-delay 10 --max-time 90 \
                 -A "Mozilla/5.0" "$url" -o "$tmp" \
               && [ "$(wc -c < "$tmp")" -gt 1000 ] \
               && grep -q "<svg" "$tmp" \
               && ! grep -qiE "something went wrong|error fetching|rate limit|user not found|could not fetch" "$tmp"; then
              mv "$tmp" "assets/$out"
              echo "Updated $out"
            else
              echo "::warning::Could not refresh $out - keeping previous version"
              rm -f "$tmp"
            fi
          }

          fetch "https://streak-stats.demolab.com/?user=${USER_NAME}&theme=radical&hide_border=true&background=0D1117&ring=6366F1&fire=6366F1" streak.svg
          fetch "https://github-readme-stats.vercel.app/api?username=${USER_NAME}&show_icons=true&theme=radical&hide_border=true&bg_color=0d1117&title_color=6366F1&icon_color=6366F1" stats.svg
          fetch "https://github-readme-stats.vercel.app/api/top-langs/?username=${USER_NAME}&layout=compact&theme=radical&hide_border=true&bg_color=0d1117&title_color=6366F1" top-langs.svg
          fetch "https://github-readme-activity-graph.vercel.app/graph?username=${USER_NAME}&theme=radical&bg_color=0d1117&color=6366F1&line=6366F1&point=ffffff&hide_border=true" activity.svg

      - name: Commit changes
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add assets
          if git diff --cached --quiet; then
            echo "No changes to commit"
          else
            git commit -m "chore: refresh profile stat cards"
            git pull --rebase origin "${GITHUB_REF_NAME}"
            git push
          fi
