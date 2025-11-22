mkdir -p .github/workflows
cat > .github/workflows/eas-build-submit.yml <<'YAML'
# (paste the exact YAML above between the YAML markers)
name: "EAS Build and Submit"

on:
  push:
    branches:
      - main
  workflow_dispatch: {}

jobs:
  build-and-submit:
    name: Build and submit (EAS)
    runs-on: ubuntu-latest
    env:
      EXPO_CLI_FORCE: "true"
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install deps
        run: |
          npm ci

      - name: Install eas-cli
        run: npm install -g eas-cli@latest

      - name: Restore secrets to files
        run: |
          mkdir -p ./credentials
          if [ -n "${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}" ]; then
            echo "${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}" > ./credentials/play-service-account.json
          fi
          if [ -n "${{ secrets.ASC_KEY }}" ]; then
            echo "${{ secrets.ASC_KEY }}" > ./credentials/AuthKey_${{ secrets.ASC_KEY_ID }}.p8
          fi
        shell: bash

      - name: Export env vars (for EAS submit)
        run: |
          if [ -f ./credentials/play-service-account.json ]; then
            echo "GOOGLE_APPLICATION_CREDENTIALS=./credentials/play-service-account.json" >> $GITHUB_ENV
          fi
          if [ -f ./credentials/AuthKey_${{ secrets.ASC_KEY_ID }}.p8 ]; then
            echo "ASC_KEY_PATH=./credentials/AuthKey_${{ secrets.ASC_KEY_ID }}.p8" >> $GITHUB_ENV
            echo "ASC_KEY_ID=${{ secrets.ASC_KEY_ID }}" >> $GITHUB_ENV
            echo "ASC_ISSUER_ID=${{ secrets.ASC_ISSUER_ID }}" >> $GITHUB_ENV
          fi
          # RevenueCat keys injected as runtime env
          echo "REVENUECAT_APPLE_API_KEY=${{ secrets.REVENUECAT_APPLE_API_KEY }}" >> $GITHUB_ENV
          echo "REVENUECAT_GOOGLE_API_KEY=${{ secrets.REVENUECAT_GOOGLE_API_KEY }}" >> $GITHUB_ENV

      - name: Login to EAS
        env:
          EAS_TOKEN: ${{ secrets.EAS_TOKEN }}
        run: |
          eas whoami || eas login --token "$EAS_TOKEN"
        shell: bash

      - name: Build Android (AAB) - production profile
        run: |
          eas build --platform android --profile production --non-interactive
        env:
          EXPO_NO_VERSIONS_CHECK: 1

      - name: Build iOS (Archive) - production profile
        run: |
          eas build --platform ios --profile production --non-interactive
        env:
          EXPO_NO_VERSIONS_CHECK: 1

      - name: Submit to Google Play (latest)
        run: |
          eas submit --platform android --latest --non-interactive
        env:
          GOOGLE_APPLICATION_CREDENTIALS: ${{ env.GOOGLE_APPLICATION_CREDENTIALS }}

      - name: Submit to App Store Connect (latest)
        run: |
          eas submit --platform ios --latest --non-interactive
        env:
          ASC_KEY_PATH: ${{ env.ASC_KEY_PATH }}
          ASC_KEY_ID: ${{ env.ASC_KEY_ID }}
          ASC_ISSUER_ID: ${{ env.ASC_ISSUER_ID }}
YAML

git add .github/workflows/eas-build-submit.yml
git commit -m "Add EAS build & submit workflow"
git push origin main
