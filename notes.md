# Github actions

## github actions for helm and kustomize workflows
```yaml
name: PR Build

on:
  pull_request:
    branches:
      - main

jobs:
  lint:
    runs-on: <image-name>
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Install missing packages
        run: |
          sudo apt update \
          && sudo apt install git curl -y \
          && curl -s https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash \
          && curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash \
          && chmod a+x kustomize \
          && sudo mv kustomize /usr/local/bin/kustomize

      - name: Kustomize Build
        id: kustomize
        run: |
          #!/bin/bash

          # set -e  # Exit immediately if a command fails
          errors=""

          # Find all directories containing a kustomization.yaml file
          directories=$(find . -type f -name "kustomization.yaml" -exec dirname {} \;)

          # Iterate over each directory and run `kustomize build`
          for dir in $directories; do
            if [ "$(basename "$dir")" != "base" ]; then
              echo "Running kustomize build for $dir"
              if ! kustomize build "$dir"; then
                echo "Error: kustomize build failed in $dir"
                errors+="\nError: kustomize build failed in $dir"
              fi
            fi
          done

          echo "::set-output name=errors::${errors}"

      - name: Helm Lint
        id: helm
        run: |
          #!/bin/bash

          errors=""
          directories=$(find . -type f -name "Chart.yaml" -exec dirname {} \;)

          for dir in $directories; do
            echo "Running Helm lint for $dir"
            if ! helm lint "$dir"; then
              echo "Error: Helm lint failed in $dir"
              errors+="\nError: Helm lint failed in $dir"
            fi
          done;

          echo "::set-output name=errors::${errors}"

      - name: Comment PR
        if: steps.kustomize.outputs.errors != '' || steps.helm.outputs.errors != ''
        uses: actions/github-script@v6
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const kustomizeErrors = '${{ steps.kustomize.outputs.errors }}';
            const helmErrors = '${{ steps.helm.outputs.errors }}';
            if (kustomizeErrors) {
              github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: `:bangbang: Kustomize build failed:\n${kustomizeErrors}`
              })
            }
            if (helmErrors) {
              github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: `:bangbang: Helm lint failed:\n${helmErrors}`
              })
            }

      - name: Fail PR
        if: steps.kustomize.outputs.errors != '' || steps.helm.outputs.errors != ''
        uses: actions/github-script@v6
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            core.setFailed('Validation failed')

      - name: Jenkins
        if: always()
        uses: LouisBrunner/checks-action@v1.6.0
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          name: Jenkins
          conclusion: ${{ job.status }}
          output: |
            {"summary":"${{ steps.test.outputs.summary }}"}

```