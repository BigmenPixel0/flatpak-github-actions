# Flatpak GitHub Actions

Build and deploy your Flatpak application using GitHub Actions

## How to use

### Building stage

Add a new workflow by creating a `.yml` file under `.github/workflows` with this content

```yaml
on:
  push:
    branches: [main]
  pull_request:
name: CI
jobs:
  flatpak:
    name: "Flatpak"
    runs-on: ubuntu-latest
    container:
      image: ghcr.io/flathub-infra/flatpak-github-actions:gnome-44
      options: --privileged
    steps:
    - uses: actions/checkout@v4
    - uses: flathub-infra/flatpak-github-actions/flatpak-builder@master
      with:
        bundle: palette.flatpak
        manifest-path: org.gnome.zbrown.Palette.yml
        cache-key: flatpak-builder-${{ github.sha }}
```

#### Inputs

| Name | Description | Required | Default |
| ---     | ----------- | ----------- |----|
| `manifest-path` | The relative path of the manifest file  | Required | - |
| `stop-at-module` | Stop at the specified module, ignoring it and all the following ones. Using this option disables generating bundles. | Optional | Build all modules from the manifest file |
| `bundle` | The bundle name  | Optional | `app.flatpak` |
| `build-bundle` | Whether to build a bundle or not | Optional | `true` |
| `repository-name` | The repository name, used to fetch the runtime when the user download the Flatpak bundle or when building the application  | Optional | `flathub` |
| `repository-url` | The repository url, used to fetch the runtime when the user download the Flatpak bundle or when building the application  | Optional | `https://flathub.org/repo/flathub.flatpakrepo` |
| `run-tests` | Enable/Disable running tests. This overrides the `flatpak-builder` option of the same name, which invokes `make check` or `ninja test`. Network and X11 access is enabled, with a display server provided by `xvfb-run`.  | Optional | `false` |
| `branch` | The default flatpak branch  | Optional | `master` |
| `cache` | Enable/Disable caching `.flatpak-builder` directory | Optional | `true` |
| `restore-cache` | Enable/Disable cache restoring. If caching is enabled. | Optional | `true` |
| `cache-key` | Specifies the cache key. CPU arch is automatically added, so there is no need to add it to the cache key. | Optional | `flatpak-builder-${sha256(manifestPath)}` |
| `arch` | Specifies the CPU architecture to build for | Optional | `x86_64` |
| `mirror-screenshots-url` | Specifies the URL to mirror screenshots | Optional | - |
| `gpg-sign` | The key to sign the package | Optional | - |
| `verbose` | Enable verbosity | Optional | `false` |
| `upload-artifact` | Whether to upload the resulting bundle or not as an artifact | Optional | `true` |

#### Building for multiple CPU architectures

To build for CPU architectures other than `x86_64`, the GitHub Actions workflow has to either natively be running on that architecture (e.g. on an `aarch64` self-hosted GitHub Actions runner), or the container used must be configured to emulate the requested architecture (e.g. with QEMU).

For example, to build a Flatpak for both `x86_64` and `aarch64` using emulation, use the following workflow as a guide:

```yaml
on:
  push:
    branches: [main]
  pull_request:
name: CI
jobs:
  flatpak:
    name: "Flatpak"
    runs-on: ubuntu-latest
    container:
      image: ghcr.io/flathub-infra/flatpak-github-actions:gnome-44
      options: --privileged
    strategy:
      matrix:
        arch: [x86_64, aarch64]
      # Don't fail the whole workflow if one architecture fails
      fail-fast: false
    steps:
    - uses: actions/checkout@v4
    # Docker is required by the docker/setup-qemu-action which enables emulation
    - name: Install deps
      if: ${{ matrix.arch != 'x86_64' }}
      run: |
        dnf -y install docker
    - name: Set up QEMU
      if: ${{ matrix.arch != 'x86_64' }}
      id: qemu
      uses: docker/setup-qemu-action@v2
      with:
        platforms: arm64
    - uses: flathub-infra/flatpak-github-actions/flatpak-builder@master
      with:
        bundle: palette.flatpak
        manifest-path: org.gnome.zbrown.Palette.yml
        cache-key: flatpak-builder-${{ github.sha }}
        arch: ${{ matrix.arch }}
```

#### Building for Automated Tests

As described in the [Inputs](#inputs) documentation, specifying `run-tests: true` will amend the Flatpak manifest to enable Network and X11 access automatically. Any other changes to the manifest must be made manually, such as building the tests (e.g. `-Dtests=true`) or any other options (e.g. `--buildtype=debugoptimized`).

Most developers will want to run tests on pull requests, before merging into the main branch of a repository. To ensure your manifest is building the correct code, you should set the `sources` entry for your project to `"type": "dir"` with the `path` key relative to the manifest's location in the repository.

In the example below, the manifest is located at `/build-aux/flatpak/org.gnome.zbrown.Palette.json`, so the `path` key is set to `../../`. If the manifest were in the project root instead, the correct usage would be `"path": "./"`.

```json
{
    "app-id" : "org.gnome.zbrown.Palette",
    "runtime" : "org.gnome.Platform",
    "runtime-version" : "master",
    "sdk" : "org.gnome.Sdk",
    "command" : "org.gnome.zbrown.Palette",
    "finish-args" : [
        "--share=ipc",
        "--device=dri",
        "--socket=fallback-x11",
        "--socket=wayland"
    ],
    "modules" : [
        {
            "name" : "palette",
            "buildsystem" : "meson",
            "config-opts" : [
                "--prefix=/app",
                "--buildtype=debugoptimized",
                "-Dtests=true"
            ],
            "sources" : [
                {
                    "name" : "palette",
                    "buildsystem" : "meson",
                    "type" : "dir",
                    "path" : "../../"
                }
            ]
        }
    ]
}
```

### Deployment stage

If you want to deploy the successfully built Flatpak application to a remote repository

```yaml
on:
  push:
    branches: [main]
name: Deploy
jobs:
  flatpak:
    name: "Flatpak"
    runs-on: ubuntu-latest
    container:
      image: ghcr.io/flathub-infra/flatpak-github-actions:gnome-44
      options: --privileged
    steps:
    - uses: actions/checkout@v4
    - uses: flathub-infra/flatpak-github-actions/flatpak-builder@master
      name: "Build"
      with:
        bundle: palette.flatpak
        manifest-path: org.gnome.zbrown.Palette.yml
        cache-key: flatpak-builder-${{ github.sha }}
    - uses: flathub-infra/flatpak-github-actions/flat-manager@master
      name: "Deploy"
      with:
        repository: elementary
        flat-manager-url: https://flatpak-api.elementary.io
        token: some_very_hidden_token
        end-of-life: "The application has been renamed to..."
        end-of-life-rebase: "org.zbrown.Palette"
```

#### Inputs

| Name | Description | Required | Default |
| ---     | ----------- | ----------- |----|
| `repository` | The repository to push the build into  | Required | - |
| `flat-manager-url` | The flat-manager remote URL  | Required | - |
| `token` | A flat-manager token  | Required | - |
| `end-of-life` | Reason for end of life  | Optional | - |
| `end-of-life-rebase` | The new app-id  | Optional | - |
| `build-log-url` | URL to Flatpak build log | Optional | - |
| `verbose` | Enable verbosity | Optional | `false` |

### Docker Image

The Docker image used for the action consists of 2 parts: The base image, based on Fedora and which can be found
[here](./Dockerfile), and the specific image of the runtime you choose, which is generated through
[this](.github/workflows/docker.yml) GitHub Actions workflow.

You can specify the specific runtime you need to use through the image tags:

| Runtime         | Version | Tag                 | Example                                                          |
| --------------- | ------- | ------------------- | ---------------------------------------------------------------- |
| Freedesktop SDK | 22.08   | `freedesktop-22.08` | `image: ghcr.io/flathub-infra/flatpak-github-actions:freedesktop-22.08` |
| Freedesktop SDK | 23.08   | `freedesktop-23.08` | `image: ghcr.io/flathub-infra/flatpak-github-actions:freedesktop-23.08` |
| GNOME           | 44    | `gnome-44`        | `image: ghcr.io/flathub-infra/flatpak-github-actions:gnome-44`        |
| GNOME           | 45    | `gnome-45`        | `image: ghcr.io/flathub-infra/flatpak-github-actions:gnome-45`        |
| GNOME           | 46    | `gnome-46`        | `image: ghcr.io/flathub-infra/flatpak-github-actions:gnome-46`        |
| KDE             | 5.15-22.08    | `kde-5.15-22.08`          | `image: ghcr.io/flathub-infra/flatpak-github-actions:kde-5.15-22.08`          |
| KDE             | 5.15-23.08    | `kde-5.15-23.08`          | `image: ghcr.io/flathub-infra/flatpak-github-actions:kde-5.15-23.08`          |
| KDE             | 6.5     | `kde-6.5`          | `image: ghcr.io/flathub-infra/flatpak-github-actions:kde-6.5`          |
| KDE             | 6.6     | `kde-6.6`          | `image: ghcr.io/flathub-infra/flatpak-github-actions:kde-6.6`          |
| KDE             | 6.7     | `kde-6.7`          | `image: ghcr.io/flathub-infra/flatpak-github-actions:kde-6.7`          |
