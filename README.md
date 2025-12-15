# build images action

Builds docker images.

## Examples

### Building two images nginx and server

Dockerfiles must be located in folders with the same name in the `docker` directory. In other words, the following structure
```
./docker/server/Dockerfile
./docker/nginx/Dockerfile
```
The `server` image also needs to pass the `PLATFORM` and `ENV` arguments. The resulting image names will be formed according to the following principle: `<registry>/<repo-name>/<image-name>:<tag>`, where `<repo-name>` is the repository name in lowercase.

```yaml
# nosemgrep
- uses: identw/build-images-action@main
  id: build-images
  with:
    registry: ${{ vars.REGISTRY }}
    registry-user: ${{ secrets.REGISTRY_USER }}
    registry-password: ${{ secrets.REGISTRY_PASSWORD }}
    tag: ${{ inputs.area }}-{{ dateTime }}-{{ ref }}-{{ commit }}
    operation: build-and-push
    build-opts: |
      - name: server
        args:
          - name: PLATFORM
            value: ${{ inputs.platform }}
          - name: ENV
            value: ${{ inputs.env }}
      - name: nginx
```

The tag field supports basic templating ([details](#tag))

If the `image-naming-strategy: single-repo` option is specified, image names will be generated according to the following pattern: `<registry>/<repo-name>:<image-name>-<tag>`:
```yaml
# nosemgrep
- uses: identw/build-images-action@main
  id: build-images
  with:
    registry: ghcr.io
    registry-user: ${{ github.repository_owner }}
    registry-password: ${{ secrets.GITHUB_TOKEN }}
    tag: ${{ inputs.area }}-{{ dateTime }}-{{ ref }}-{{ commit }}
    operation: build-and-push
    build-opts: |
      - name: server
        args:
          - name: PLATFORM
            value: ${{ inputs.platform }}
          - name: ENV
            value: ${{ inputs.env }}
      - name: nginx
```
The following images will be built:  
1. `ghcr.io/org-name/repo-name:server-<tag>`
1. `ghcr.io/org-name/repo-name:nginx-<tag>`

### Separating build and push steps to registry and copying files from images

Sometimes it's necessary to copy a file from a built image and perform some actions with it:
```yaml
# nosemgrep
- uses: identw/build-images-action@main
  id: build-images
  with:
    registry: ${{ vars.REGISTRY }}
    registry-user: ${{ secrets.REGISTRY_USER }}
    registry-password: ${{ secrets.REGISTRY_PASSWORD }}
    tag: ${{ inputs.area }}-{{ dateTime }}-{{ ref }}-{{ commit }}
    operation: build
    build-opts: |
      - name: server
        copy-files: ['/app/junit.xml']
        args:
          - name: PLATFORM
            value: ${{ inputs.platform }}
          - name: ENV
            value: ${{ inputs.env }}
      - name: nginx-server

- name: check copy files
  run: |
    files="${{ join(fromJSON(steps.build-images.outputs.copy-files), ' ') }}"
    for i in $files;
    do
      cat $i
    done

# nosemgrep
- uses: identw/build-images-action@main
  id: push-images
  with:
    registry: ${{ vars.REGISTRY }}
    registry-user: ${{ secrets.REGISTRY_USER }}
    registry-password: ${{ secrets.REGISTRY_PASSWORD }}
    tag: ${{ inputs.area }}-{{ dateTime }}-{{ ref }}-{{ commit }}
    operation: push
    build-opts: ${{ steps.build-images.outputs.build-opts }}
```

### Building multiple images from one Dockerfile but with different targets
```yaml
# nosemgrep
- uses: identw/build-images-action@main
  id: build-images
  with:
    registry: ${{ vars.REGISTRY }}
    registry-user: ${{ secrets.REGISTRY_USER }}
    registry-password: ${{ secrets.REGISTRY_PASSWORD }}
    tag: ${{ inputs.area }}-{{ dateTime }}-{{ ref }}-{{ commit }}
    operation: build-and-push
    build-opts: |
      - name: php
        file: ./docker/Dockerfile
        target: php
      - name: migrations
        file: ./docker/Dockerfile
        target: migrations
```

### If you need to run a built container after the build
The action returns an output `built-images`, which can be used to run just-built containers:
```yaml
- uses: identw/build-images-action@main
  id: build-images
  with:
    registry: ${{ vars.REGISTRY }}
    registry-user: ${{ secrets.REGISTRY_USER }}
    registry-password: ${{ secrets.REGISTRY_PASSWORD }}
    tag: latest
    operation: build
    build-opts: |
      - name: native-builder
- name: test
  env:
    IMAGE: ${{ fromJson(steps.build-images.outputs.built-images).native-builder }}
  run: |
    docker run --rm -v $(pwd):/app --workdir /app ${IMAGE} ./build.sh
```

### Passing secrets
If we want to use the https://docs.docker.com/build/building/secrets/ functionality, we need to pass the `secrets` field

Docker secrets supports the ability to pass secrets through environment variables, so there is an `envs` field that allows creating environment variables before running `docker build` to pass them to secrets with `type=env`
```yaml
# nosemgrep
- uses: identw/build-images-action@main
  id: build-images
  with:
    registry: ghcr.io
    registry-user: ${{ github.repository_owner }}
    registry-password: ${{ secrets.GITHUB_TOKEN }}
    tag: ${{ inputs.area }}-{{ dateTime }}-{{ ref }}-{{ commit }}
    operation: build-and-push
    build-opts: |
      - name: server
        envs:
        - name: GITHUB_USER
          value: ${{ github.repository_owner }}
        - name: GITHUB_TOKEN
          value: ${{ secrets.COMMON_TOKEN }}
        secrets:
        - id=GITHUB_USER,type=env,env=GITHUB_USER
        - id=GITHUB_TOKEN,type=env,env=GITHUB_TOKEN
```

```dockerfile
RUN --mount=type=secret,id=GITHUB_USER,env=GITHUB_USER \
    --mount=type=secret,id=GITHUB_TOKEN,env=GITHUB_TOKEN \
    make build
```
Second example

```yaml
- uses: identw/build-images-action@main
  name: build native
  id: build-images
  with:
    registry: ${{ vars.REGISTRY }}
    registry-user: ${{ secrets.REGISTRY_USER }}
    registry-password: ${{ secrets.REGISTRY_PASSWORD }}
    tag: ${{ inputs.area }}-{{ ref }}
    operation: build
    build-opts: |
      - name: android
        copy-files: ['/output']
        envs:
        - name: ANDROID_KEYSTORE_PASSWORD
          value: ${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
        - name: ANDROID_KEYSTORE_ALIAS
          value: ${{ secrets.ANDROID_KEYSTORE_ALIAS }}
        secrets:
        - id=ANDROID_KEYSTORE,type=file,src=./android.keystore
        - id=ANDROID_KEYSTORE_PASSWORD,type=env,env=ANDROID_KEYSTORE_PASSWORD
        - id=ANDROID_KEYSTORE_ALIAS,type=env,env=ANDROID_KEYSTORE_ALIAS
        - id=SUPPLY_JSON_KEY,type=file,src=./google_play_key.json
        args:
        - name: AREA
          value: ${{ inputs.area }}
        - name: VERSION_NAME
          value: ${{ github.ref_name }}
        - name: BACKEND_URL
          value: ${{ inputs.backend-url }}
```
In the example, we pass the secrets `ANDROID_KEYSTORE`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEYSTORE_ALIAS`, `SUPPLY_JSON_KEY`, where `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEYSTORE_ALIAS` are passed through environment variables, and `ANDROID_KEYSTORE`, `SUPPLY_JSON_KEY` are passed through files. 
In the Dockerfile, secrets can be used as follows

```dockerfile
RUN --mount=type=secret,id=ANDROID_KEYSTORE \
    --mount=type=secret,id=ANDROID_KEYSTORE_PASSWORD,env=ANDROID_KEYSTORE_PASSWORD \
    --mount=type=secret,id=ANDROID_KEYSTORE_ALIAS,env=ANDROID_KEYSTORE_ALIAS \
    --mount=type=secret,id=SUPPLY_JSON_KEY \
       export ANDROID_KEYSTORE=/run/secrets/ANDROID_KEYSTORE \
    && export SUPPLY_JSON_KEY=/run/secrets/SUPPLY_JSON_KEY \
    make build
```
See more details in the docker secrets documentation: https://docs.docker.com/build/building/secrets/

### Specifying multiple platforms for build
specify `platforms` for building all images
```yaml
- uses: identw/build-images-action@main
  id: build-images
  with:
    registry: ${{ vars.REGISTRY }}
    registry-user: ${{ secrets.REGISTRY_USER }}
    registry-password: ${{ secrets.REGISTRY_PASSWORD }}
    tag: latest
    operation: build-and-push
    platforms: linux/amd64,linux/arm64
    build-opts: |
      - name: image
```

specify `platforms` for a specific image
```yaml
- uses: identw/build-images-action@main
  id: build-images
  with:
    registry: ${{ vars.REGISTRY }}
    registry-user: ${{ secrets.REGISTRY_USER }}
    registry-password: ${{ secrets.REGISTRY_PASSWORD }}
    tag: build-and-push
    operation: build
    build-opts: |
      - name: image1
        platforms: linux/amd64,linux/arm64
      - name: image2
```

### latest tag
Sometimes you want to push an additional latest tag along with the specified tag. To do this, simply add the `latest: true` parameter

```yaml
# nosemgrep
- uses: identw/build-images-action@main
  id: build-images
  with:
    registry: ${{ vars.REGISTRY }}
    registry-user: ${{ secrets.REGISTRY_USER }}
    registry-password: ${{ secrets.REGISTRY_PASSWORD }}
    tag: ${{ inputs.area }}-{{ dateTime }}-{{ ref }}-{{ commit }}
    operation: build-and-push
    latest: true
    build-opts: |
      - name: server
        args:
          - name: PLATFORM
            value: ${{ inputs.platform }}
          - name: ENV
            value: ${{ inputs.env }}
      - name: nginx
```

You can also do the same at the image level in `build-opts`
```yaml
# nosemgrep
- uses: identw/build-images-action@main
  id: build-images
  with:
    registry: ${{ vars.REGISTRY }}
    registry-user: ${{ secrets.REGISTRY_USER }}
    registry-password: ${{ secrets.REGISTRY_PASSWORD }}
    tag: ${{ inputs.area }}-{{ dateTime }}-{{ ref }}-{{ commit }}
    operation: build-and-push
    build-opts: |
      - name: server
        args:
          - name: PLATFORM
            value: ${{ inputs.platform }}
          - name: ENV
            value: ${{ inputs.env }}
      - name: nginx
        latest: true
```

### cache
You can use cache-from and cache-to

```yaml
# nosemgrep
- uses: identw/build-images-action@main
  id: build-images
  with:
    registry: ${{ vars.REGISTRY }}
    registry-user: ${{ secrets.REGISTRY_USER }}
    registry-password: ${{ secrets.REGISTRY_PASSWORD }}
    tag: ${{ inputs.area }}-{{ dateTime }}-{{ ref }}-{{ commit }}
    operation: build-and-push
    cache-from: type=gha
    cache-to: type=gha,mode=max
    build-opts: |
      - name: server
        args:
          - name: PLATFORM
            value: ${{ inputs.platform }}
          - name: ENV
            value: ${{ inputs.env }}
      - name: nginx
```

If the `image-naming-strategy: single-repo` option is used, then in the case of multiple images, the `latest` tag will be pushed as follows: `ghcr.io/<org-name>/<repo-name>:<image-name>-latest`

### image = repository name

To push an image with the repository name without additional suffixes, i.e., in the following way
1. `<registry>/<repo-name>:<tag>`
1. `ghcr.io/<org-name>/<repo-name>:<tag>`

you can use the `repo-image-name` option

```yaml
# nosemgrep
- uses: identw/build-images-action@main
  id: build-images
  with:
    registry: ${{ vars.REGISTRY }}
    registry-user: ${{ secrets.REGISTRY_USER }}
    registry-password: ${{ secrets.REGISTRY_PASSWORD }}
    tag: ${{ inputs.area }}-{{ dateTime }}-{{ ref }}-{{ commit }}
    operation: build-and-push
    build-opts: |
      - name: server
        repo-image-name: true
        args:
          - name: PLATFORM
            value: ${{ inputs.platform }}
          - name: ENV
            value: ${{ inputs.env }}
      - name: nginx
```

The following images will be built

1. `<registry>/<repo-name>:<tag>` - server
1. `<registry>/<repo-name>/nginx:<tag>` - nginx

### changing the repository name
Usually, the image name is formed according to the principle: `<registry>/<repo-name>/<image-name>:<tag>`, where `<repo-name>` is the repository name in lowercase. If you pass the `repo-name` parameter, this part of the name will be replaced with the specified value (also in lowercase).

```yaml
- uses: identw/build-images-action@main
  id: build-images
  with:
    registry: ${{ vars.REGISTRY }}
    registry-user: ${{ secrets.REGISTRY_USER }}
    registry-password: ${{ secrets.REGISTRY_PASSWORD }}
    tag: latest
    operation: build-and-push
    repo-name: override/repo-name
    build-opts: |
      - name: server
```
As a result, the following name will be formed: `<registry>/override/repo-name/server:latest`

If the standard registry `ghcr.io` is used, this option has no effect. `repo-name` will be equal to the repository name.

The parameter supports basic templating:
```yaml
- uses: identw/build-images-action@main
  id: build-images
  with:
    registry: ${{ vars.REGISTRY }}
    registry-user: ${{ secrets.REGISTRY_USER }}
    registry-password: ${{ secrets.REGISTRY_PASSWORD }}
    tag: latest
    operation: build-and-push
    repo-name: '{{ repo }}/override'
    build-opts: |
      - name: server
```

As a result, the following name will be formed: `<registry>/repo-name/override/server:latest`


### CI

Often there is a task to simply build images for CI checks. They usually don't need to be pushed and there's no need to set special tags. Therefore, you have to add logic to the workflow for different launch modes and specify different tags depending on input parameters. For example:

```yaml
    - name: vars
      id: vars
      run: |
        set -x
        commit_sha=${{ github.sha }}
        commit_sha=${commit_sha:0:10}
        echo "commit_sha=${commit_sha}" >> $GITHUB_OUTPUT
        time=`date -u +%Y%m%d%H%M`

        platform="${{ inputs.app_platform }}"
        if [[ ${{ inputs.custom_tag_platform }} != 'default' ]]; then
          platform="${{ inputs.custom_tag_platform }}"
        fi

        tag="${{ inputs.app_env }}-manual-${time}-${{ github.ref_name }}-${commit_sha}"
        operation='build-and-push'

        if [[ ${{ inputs.ci }} == "true" ]]; then
          tag="${commit_sha}"
          operation='build'
        fi

        echo "operation=${operation}" >> $GITHUB_OUTPUT
        echo "tag=${tag}" >> $GITHUB_OUTPUT
        echo "platform=${platform}" >> $GITHUB_OUTPUT


    # nosemgrep
    - uses: identw/build-images-action@main
      id: build-images
      with:
        registry: ${{ vars.REGISTRY }}
        registry-user: ${{ secrets.REGISTRY_USER }}
        registry-password: ${{ secrets.REGISTRY_PASSWORD }}
        tag: ${{ steps.vars.outputs.tag }}
        operation: ${{ steps.vars.outputs.operation }}
        repo-name: '{{ repo }}/${{ steps.vars.outputs.platform }}'
        build-opts: |
          - name: server
          - name: web
          - name: remote
          - name: fbinstant-deploy
            args:
              - name: AREA
                value: ${{ inputs.app_env }}
              - name: REF
                value: ${{ github.ref_name }}
```
But this can be simplified by simply setting the `ci` parameter to `true`:

```yaml
    - name: vars
      id: vars
      run: |
        set -x
        platform="${{ inputs.app_platform }}"
        if [[ ${{ inputs.custom_tag_platform }} != 'default' ]]; then
          platform="${{ inputs.custom_tag_platform }}"
        fi

        echo "platform=${platform}" >> $GITHUB_OUTPUT

    # nosemgrep
    - uses: identw/build-images-action@main
      id: build-images
      with:
        registry: ${{ vars.REGISTRY }}
        registry-user: ${{ secrets.REGISTRY_USER }}
        registry-password: ${{ secrets.REGISTRY_PASSWORD }}
        ci: ${{ inputs.ci }}
        tag: '${{ inputs.app_env }}-manual-{{ dateTime }}-${{ ref }}-{{ commit }}'
        operation: build-and-push
        repo-name: '{{ repo }}/${{ steps.vars.outputs.platform }}'
        build-opts: |
          - name: server
          - name: web
          - name: remote
          - name: fbinstant-deploy
            args:
              - name: AREA
                value: ${{ inputs.app_env }}
              - name: REF
                value: ${{ github.ref_name }}
```
when we set `ci` to `true`, the action doesn't push images (i.e., operation is essentially equal to `build`), and the tag will be equal to the short sha of the commit. The tag can be changed via the `ci-tag` option.

### arguments support templating

Just like in tags, you can use templates in image build arguments

```yaml
    # nosemgrep
    - uses: identw/build-images-action@main
      id: build-images
      with:
        registry: ${{ vars.REGISTRY }}
        registry-user: ${{ secrets.REGISTRY_USER }}
        registry-password: ${{ secrets.REGISTRY_PASSWORD }}
        tag: '${{ inputs.app_env }}-manual-{{ dateTime }}-${{ ref }}-{{ commit }}'
        operation: build-and-push
        build-opts: |
          - name: fbinstant-deploy
            args:
              - name: AREA
                value: ${{ inputs.app_env }}
              - name: REF
                value: '{{ ref }}'
              - name: REF
                value: '{{ commit }}'
```



## Inputs

### `registry`
Registry, specify without protocol (for example `example.com/registry`)

### `registry-user`
User for authentication in the registry

### `registry-password`
Password for authentication in the registry

### `tag`
Image tag. Supports basic templating:

1. `{{ commit }}` - short sha of the commit
1. `{{ dateTime }}` - date and time in UTC in `YYYYMMDDhhmm` format
1. `{{ ref }}` - branch or tag name (without refs/heads)
1. `{{ pr }}` - if this is a build triggered by a PR event, the request number is substituted, otherwise `manual`

Example: `tag: '${{ inputs.area }}-{{ dateTime }}-{{ ref }}-{{ commit }}'`


### `operation`
Can be equal to `build`, `push`, `build-and-push`. If equal to `build`, images will be built but not pushed to the registry. If `push`, the action will simply push images (it is expected that images are built for the specified tag). `build-and-push` immediately builds images and pushes them

### `platforms`
`default: ''`
List of platforms that will be substituted into the build command via the `--platform` argument. Platforms should be comma-separated. Example: `linux/amd64,linux/arm64`. Documentation https://docs.docker.com/build/building/multi-platform/. This parameter can be overridden at the image level in `build-opts`.

### `latest`
`default: false`
Whether to additionally push the latest tag or not. This parameter can be overridden at the image level in `build-opts`.

### `repo-name`
`default: ''`
Overwrites the part of the image name where the repository name is substituted: `<registry>/<repo-name>/<image-name>:<tag>`. If this option is specified, the `<repo-name>` part will be replaced. Basic templating is supported:

1. `{{ repo }}` - current repository name (without owner or organization)

If the standard registry `ghcr.io` is used, this option has no effect. `repo-name` will be equal to the repository name.

### `image-naming-strategy`
`default: 'multi-repo'`
Can take the following values: `multi-repo`, `single-repo`.
This option determines the image naming strategy:
1. `multi-repo`: `<registry>/<repo-name>/<image-name>:<tag>`
1. `single-repo`: `<registry>/<repo-name>:<image-name>:<tag>`

For `single-repo`, the `latest` tag will be set as `<registry>/<repo-name>:<image-name>-latest`

### `ci`
`default: 'false'`
If equal to `'true'`, `operation` is forcibly set to `build`, and tag is set to `ci-tag`

### `ci-tag`
`default: '{{ commit }}'`

Used instead of `tag` when `ci` is equal to `true`. Supports similar templating [details](#tag)


### `build-opts`
Accepts a data structure in yaml format of the following form:
```yaml
- name: <image1>
  target: t1
  file: ./docker/<image1>/Dockerfile
  platforms: linux/amd64,linux/arm64
  latest: false
  repo-image-name: false
  args:
    - name: arg1
      value: val2
    - name: arg2
      value: val2
    - ...
    - name: argn
      value: argn
  copy-files: ['path/to/file1', 'path/to/file2', ..., 'path/to/filen']
  envs:
    - name: VAR1
      value: VAL1
    - name: VAR2
      value: VAL2
  secrets:
    - id=FILE,type=file,src=./file
    - id=VAR1,type=env,env=VAR1
    - id=VAR2,type=env,env=VAR2

- name: <image2>:
  target: t1
  file: ./docker/<image2>/Dockerfile
  platforms: linux/amd64,linux/arm64
  latest: false
  repo-image-name: false
  args:
    - name: arg1
      value: val2
    - name: arg2
      value: val2
    - ...
    - name: argn
      value: argn
    - name: commit
      value: '{{ commit }}'
    - name: ref
      value: '{{ ref }}'
  copy-files: ['path/to/file1', 'path/to/file2', ..., 'path/to/filen']
  envs:
    - name: VAR1
      value: VAL1
    - name: VAR2
      value: VAL2
  secrets:
    - id=FILE,type=file,src=./file
    - id=VAR1,type=env,env=VAR1
    - id=VAR2,type=env,env=VAR2
...
- name: <imagen>
...
```
The structure represents an array, where each element is an image and additional parameters for it.

* `name` - image that needs to be built. 

* `args` (optional) - list of arguments. Supports templating like in tags ([details](#tag))
* `copy-files` (optional) - files that need to be copied from the image after building  
* `target` (optional) - if specified, `--target target-value` is added to the build command
* `file` (optional) - if specified, the value from this field is substituted in `--file`
* `envs` (optional) - creates specified environment variables before starting the build, useful when used together with `secrets`
* `secrets` (optional) - adds `--secret` arguments. See more details in the docker documentation: https://docs.docker.com/build/building/secrets/
* `platforms` (optional) - adds `--platform platforms` to the build arguments. Platforms should be comma-separated. Example: `linux/amd64,linux/arm64`. Documentation https://docs.docker.com/build/building/multi-platform/
* `latest` (optional) - whether to additionally push the latest tag or not
* `repo-image-name` (optional) - pushes the image under the repository name

## Outputs 

### `copy-files`

List of files in JSON format: `["path/to/file1", "path/to/file2", ...]`, paths to files that were copied from containers specified in the `copy-files` option for images in `build-opts`.

### `pushed-images`

List of images pushed to the registry in JSON format: `["example.com/registry/image1:tag", "example.com/registry/image2:tag", ...]`

### `built-images`

Built images in JSON format: `{"image1": "example.com/registry/image1:tag", "image2": "example.com/registry/image2:tag", ...}`
This is useful if you need to run a container from a built image later in the pipeline
```yaml
      - uses: identw/build-images-action@main
        id: build-images
        with:
          registry: ${{ vars.REGISTRY }}
          registry-user: ${{ secrets.REGISTRY_USER }}
          registry-password: ${{ secrets.REGISTRY_PASSWORD }}
          tag: latest
          operation: build
          build-opts: |
            - name: native-builder
      - name: test
        env:
          IMAGE: ${{ fromJson(steps.build-images.outputs.built-images).native-builder }}
        run: |
          docker run --rm -v $(pwd):/app --workdir /app ${IMAGE} ./build.sh
```


### `build-opts`

Passes to outputs the same thing that came in the input with the same name