# CircleCI/Docker Check

## CircleCI input

```yaml
orbs:
  docker: circleci/docker@x.y
jobs:
  docker-example:
    docker:
      - image: 'cimg/base:stable'
    steps:
      - docker/check:
          docker-password: DOCKER_PASSWORD
          docker-username: DOCKER_LOGIN
          registry: docker.io
          use-docker-credentials-store: true
```

### Transformed Github Action (Composite Action Feature Enabled)
workflow
```yaml
uses: "./.github/actions/docker_check"
with:
  docker-password: "${{ secrets.DOCKER_PASSWORD }}"
  docker-username: "${{ env.DOCKER_LOGIN }}"
  registry: docker.io
  use-docker-credentials-store: true
```
action.yml
```yaml
name: docker_check
inputs:
  docker-password:
    required: true
  docker-username:
    required: true
  registry:
    required: false
    default: docker.io
  use-docker-credentials-store:
    required: false
    default: false
runs:
  using: composite
  steps:
  - id: release_tag
    run: |-
      RELEASE_VERSION=$(curl -Ls --fail --retry 3 -o /dev/null -w %{url_effective} 'https://github.com/docker/docker-credential-helpers/releases/latest' | sed 's:.*/::')
      echo "release_tag=$RELEASE_VERSION" >> $GITHUB_OUTPUT
    if: "${{ fromJSON(inputs.use-docker-credentials-store) }}"
    shell: bash
  - run: |-
      HELPER_FILENAME="docker-credential-${{ env.HELPER_NAME }}"

      if which "$HELPER_FILENAME" > /dev/null 2>&1; then
        echo "$HELPER_FILENAME is already installed"
        exit 0
      fi

      GPG_TEMPLATE=$(mktemp gpg_template.XXXXXX)
      cat > $GPG_TEMPLATE <<-EOF
        Key-Type: RSA
        Key-Length: 2048
        Name-Real: GitHubActions
        Expire-Date: 0
        %no-protection
        %no-ask-passphrase
        %commit
      EOF

      if [ "$HELPER_FILENAME" = "docker-credential-pass" ]; then
        sudo apt-get update --yes && sudo apt-get install gnupg2 pass --yes
        gpg2 --batch --gen-key "$GPG_TEMPLATE"
        FINGERPRINT_STRING=$(gpg2 --list-keys --with-fingerprint --with-colons GitHubActions | grep fpr)
        arrFINGERPRINT=(${FINGERPRINT_STRING//:/ })
        FINGERPRINT=${arrFINGERPRINT[-1]}
        pass init $FINGERPRINT
      fi
      rm $GPG_TEMPLATE

      curl -L -o "${HELPER_FILENAME}_archive" "https://github.com/docker/docker-credential-helpers/releases/download/${{ env.RELEASE_TAG }}/${HELPER_FILENAME}-${{ env.RELEASE_TAG }}-amd64.tar.gz"
      tar xvf "./${HELPER_FILENAME}_archive"
      chmod +x "./$HELPER_FILENAME"
      mv "./$HELPER_FILENAME" "${{ env.BIN_PATH }}/$HELPER_FILENAME"
      "${{ env.BIN_PATH }}/$HELPER_FILENAME" version
      rm "./${HELPER_FILENAME}_archive"
    env:
      HELPER_NAME: pass
      RELEASE_TAG: "${{ steps.release_tag.outputs.release_tag }}"
      BIN_PATH: "/usr/local/bin"
    if: "${{ fromJSON(inputs.use-docker-credentials-store) }}"
    shell: bash
  - run: |-
      mkdir -p $(dirname $HOME/.docker/config.json)
      cat $HOME/.docker/config.json | jq --arg credsStore '${{ env.HELPER_NAME }}' '. + {credsStore: $credsStore}' > /tmp/docker-config-credsstore-update.json
      cat /tmp/docker-config-credsstore-update.json > $HOME/.docker/config.json
      rm /tmp/docker-config-credsstore-update.json
    env:
      HELPER_NAME: pass
    if: "${{ fromJSON(inputs.use-docker-credentials-store) }}"
    shell: bash
  - uses: docker/login-action@v2.1.0
    with:
      username: "${{ inputs.docker-username }}"
      password: "${{ inputs.docker-password }}"
      registry: "${{ inputs.registry }}"
```
### Transformed Github Action (Composite Action Feature Disabled)
```yaml
jobs:
  docker-example:
    runs-on: ubuntu-latest
    container:
      image: ubuntu
    steps:
    - id: release_tag
      run: |-
        RELEASE_VERSION=$(curl -Ls --fail --retry 3 -o /dev/null -w %{url_effective} 'https://github.com/docker/docker-credential-helpers/releases/latest' | sed 's:.*/::')
        echo "release_tag=$RELEASE_VERSION" >> $GITHUB_OUTPUT
    - run: |-
        HELPER_FILENAME="docker-credential-${{ env.HELPER_NAME }}"
        if which "$HELPER_FILENAME" > /dev/null 2>&1; then
          echo "$HELPER_FILENAME is already installed"
          exit 0
        fi
        GPG_TEMPLATE=$(mktemp gpg_template.XXXXXX)
        cat > $GPG_TEMPLATE <<-EOF
          Key-Type: RSA
          Key-Length: 2048
          Name-Real: GitHubActions
          Expire-Date: 0
          %no-protection
          %no-ask-passphrase
          %commit
        EOF
        if [ "$HELPER_FILENAME" = "docker-credential-pass" ]; then
          sudo apt-get update --yes && sudo apt-get install gnupg2 pass --yes
          gpg2 --batch --gen-key "$GPG_TEMPLATE"
          FINGERPRINT_STRING=$(gpg2 --list-keys --with-fingerprint --with-colons GitHubActions | grep fpr)
          arrFINGERPRINT=(${FINGERPRINT_STRING//:/ })
          FINGERPRINT=${arrFINGERPRINT[-1]}
          pass init $FINGERPRINT
        fi
        rm $GPG_TEMPLATE
        curl -L -o "${HELPER_FILENAME}_archive" "https://github.com/docker/docker-credential-helpers/releases/download/${{ env.RELEASE_TAG }}/${HELPER_FILENAME}-${{ env.RELEASE_TAG }}-amd64.tar.gz"
        tar xvf "./${HELPER_FILENAME}_archive"
        chmod +x "./$HELPER_FILENAME"
        mv "./$HELPER_FILENAME" "${{ env.BIN_PATH }}/$HELPER_FILENAME"
        "${{ env.BIN_PATH }}/$HELPER_FILENAME" version
        rm "./${HELPER_FILENAME}_archive"
      env:
        HELPER_NAME: pass
        RELEASE_TAG: "${{ steps.release_tag.outputs.release_tag }}"
        BIN_PATH: "/usr/local/bin"
    - run: |-
        mkdir -p $(dirname $HOME/.docker/config.json)
        cat $HOME/.docker/config.json | jq --arg credsStore '${{ env.HELPER_NAME }}' '. + {credsStore: $credsStore}' > /tmp/docker-config-credsstore-update.json
        cat /tmp/docker-config-credsstore-update.json > $HOME/.docker/config.json
        rm /tmp/docker-config-credsstore-update.json
      env:
        HELPER_NAME: pass
    - uses: docker/login-action@v2.1.0
      with:
        username: "${{ env.DOCKER_LOGIN }}"
        password: "${{ secrets.DOCKER_PASSWORD }}"
        registry: docker.io
```

### Unsupported Options

- None
