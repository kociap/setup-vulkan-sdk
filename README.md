# setup-vulkan-sdk

<!-- [![test setup-vulkan-sdk](https://github.com/kociap/setup-vulkan-sdk/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/humbletim/setup-vulkan-sdk/actions/workflows/ci.yml) -->

Build and integrate individual Vulkan SDK components directly from the Khronos
source repositories.

This action is meant to offer a more lightweight option for CI/CD compared to
installing the full Vulkan SDK. This is especially the case for projects that
only need the Vulkan headers and the Vulkan loader. The two SDK components from
Khronos source usually require around a minute to build and only approximately
20MB of disk space (compared to the 600M-1.8GB that a full SDK install would
require). The action utilises the GitHub cache mechanism to accelerate
subsequent builds.

All versions of the Vulkan SDK are supported. Any components provided by a
Vulkan SDK release may be built.

## Usage

_NOTE: For help with GitHub Actions, see GitHub Help Documentation
[Quickstart](https://docs.github.com/en/actions/quickstart) or [Creating a
workflow
file](https://docs.github.com/en/actions/using-workflows#creating-a-workflow-file)._

### Example integration

```yaml
  -name: Prepare Vulkan SDK
   uses: kociap/setup-vulkan-sdk@v1.0
   with:
     vulkan-query-version: 1.3.296.0
     vulkan-components: Vulkan-Headers, Vulkan-Loader
     vulkan-use-cache: true
```

All SDK releases are supported. SDK version numbers are resolved into
corresponding Khronos repos and commit points using the official LunarG [SDK
web API](https://vulkan.lunarg.com/content/view/latest-sdk-version-api).

It is also possible to specify `latest` and the action will build the most
recent release of the Vulkan SDK.

## Action Parameters

- **`vulkan-query-version`**: valid SDK release number (eg: `1.2.161.1` or
  `latest`). *[required]*
- **`vulkan-config-file`**: project-local config.json file path. *[optional;
  default: '']*
  - note: config.json files already contain versioning info, so when specified
    vulkan-query-version will be ignored
- **`vulkan-use-cache`**: if `true`, after a successful build the VULKAN_SDK
  folder is cached and restored in subsequent builds. *[optional;
  default=false]*
- **`vulkan-components`**: a space or comma delimited list of individual Vulkan
  component selections *[required]*:

Officially supported release numbers may be found here:
https://vulkan.lunarg.com/sdk/home

Documentation on querying config.json SDK specs may be found here:
https://vulkan.lunarg.com/content/view/latest-sdk-version-api

## Advanced integration

```yaml
  - name: Fetch Vulkan SDK version spec
    shell: bash
    run: |
      curl -o vulkan-sdk-config.json https://vulkan.lunarg.com/sdk/config/latest/linux/config.json

  - name: Configure Vulkan SDK using the downloaded spec
    uses: kociap/setup-vulkan-sdk@v1.0
    with:
      vulkan-config-file: vulkan-sdk-config.json
      vulkan-components: Vulkan-Headers, Vulkan-Loader
      vulkan-use-cache: true
```

To "lock in" the versions of the Khronos repositories (and avoid any
dependency on LunarG web services), commit a copy of the config.json(s) to
your project and then reference them in your actions, as show above.

<!-- Additional integration examples can be found as part of this project's CI test -->
<!-- suite: [.github/workflows/ci.yml](.github/workflows/ci.yml). -->

## References
- [Vulkan SDK](https://www.lunarg.com/vulkan-sdk/)
- [Vulkan SDK web services API](https://vulkan.lunarg.com/content/view/latest-sdk-version-api)
