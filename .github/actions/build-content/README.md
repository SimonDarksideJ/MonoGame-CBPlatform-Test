# Build MonoGame Content Action

A reusable composite GitHub Action for building MonoGame content using the MonoGame Content Pipeline Builder. This action provides flexibility to work with content builders and assets from local or external repositories.

## Features

- Build MonoGame content for any platform (iOS, Android, DesktopGL, Windows, etc.)
- Support for local or external content builder repositories
- Support for local or external asset repositories
- Automatic temp folder management
- Comprehensive logging with artifact uploads
- Optional content output upload
- Custom argument support for advanced scenarios

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `content-project-path` | Yes | `./CBPlatformTest/Content` | Path to the content project folder (relative to repo root). This is where the MonoGame Content Builder console application resides. |
| `content-builder-repo` | No | `''` | Optional GitHub repository for the content builder in `owner/repo` format. If provided, the repository will be cloned to the `content-project-path` location. Leave empty to use a local content builder. |
| `content-builder-subfolder` | No | `''` | Optional subfolder within the cloned content builder repository (e.g., `"MyBuilder"` or `"builders/desktop"`). Useful when a repository contains multiple builder configurations for different platforms or scenarios. |
| `assets-source-path` | Yes | `./Assets` | Path to the assets source folder (relative to repo root or content project). This contains your game's raw assets (images, sounds, fonts, etc.). |
| `assets-repo` | No | `''` | Optional GitHub repository for assets in `owner/repo` format. If provided, the repository will be cloned to the `assets-source-path` location. Leave empty to use local assets. |
| `assets-subfolder` | No | `''` | Optional subfolder within the cloned assets repository (e.g., `"DesktopAssets"` or `"assets/shared"`). Useful when a repository contains assets organized by platform or project. |
| `monogame-platform` | Yes | - | MonoGame platform target. Valid values: `iOS`, `Android`, `DesktopGL`, `Windows`, `WindowsStoreApp`, `MacOSX`, `Linux`, `PlayStation4`, `XboxOne`, `Switch`, etc. |
| `output-folder` | Yes | - | Output folder for processed content (relative to repo root). The compiled content will be placed here, ready to be included in your game build. |
| `additional-args` | No | `''` | Additional arguments to pass to the content builder CLI (e.g., `--verbose` or `--rebuild`). Arguments should be space-separated. |
| `upload-output` | No | `false` | Whether to upload the content output as a GitHub artifact. Set to `true` to enable. Useful for debugging or distributing pre-built content. |

## Outputs

| Output | Description |
|--------|-------------|
| `output-folder` | Full absolute path to the output folder containing the processed content. |
| `log-file` | Full absolute path to the content pipeline build log file. |
| `success` | Boolean indicating whether the content build succeeded (`true` or `false`). |

## Usage Examples

### Example 1: Content Builder and Assets in Game Project

All components (game, content builder, and assets) are in the same repository.

```yaml
name: Build Game with Content

on:
  workflow_dispatch

jobs:
  build:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v5
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v5
        with:
          dotnet-version: '9.0.x'
      
      - name: Process content
        uses: ./.github/actions/build-content
        with:
          content-project-path: './Content'
          assets-source-path: './Content/Assets'
          monogame-platform: 'DesktopGL'
          output-folder: './MyGame/bin/Release/Content'
      
      - name: Build game
        run: dotnet build -c Release MyGame/MyGame.csproj -r win-x64
```

### Example 2: Content Builder and Game Local, Assets from External Repository

The content builder and game are in the main repository, but assets are stored in a separate repository for sharing across projects.

```yaml
name: Build with External Assets

on:
  workflow_dispatch

jobs:
  build:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v5
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v5
        with:
          dotnet-version: '9.0.x'
      
      - name: Process content
        uses: ./.github/actions/build-content
        with:
          content-project-path: './Content'
          assets-source-path: './SharedAssets'
          assets-repo: 'MyOrg/game-assets'
          monogame-platform: 'DesktopGL'
          output-folder: './MyGame/bin/Release/Content'
      
      - name: Build game
        run: dotnet build -c Release MyGame/MyGame.csproj -r win-x64
```

### Example 3: Content Builder and Assets in One Repository, Game in Another

The content builder and assets are maintained in a shared tools repository, while the game code is in its own repository.

**In the game repository:**

```yaml
name: Build Game

on:
  workflow_dispatch

jobs:
  build:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v5
      
      - name: Checkout content builder action
        uses: actions/checkout@v5
        with:
          repository: MyOrg/monogame-content-tools
          path: .github/actions/build-content
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v5
        with:
          dotnet-version: '9.0.x'
      
      - name: Process content
        uses: ./.github/actions/build-content
        with:
          content-project-path: './ContentBuilder'
          content-builder-repo: 'MyOrg/monogame-content-tools'
          assets-source-path: './ContentBuilder/Assets'
          monogame-platform: 'DesktopGL'
          output-folder: './MyGame/bin/Release/Content'
      
      - name: Build game
        run: dotnet build -c Release MyGame/MyGame.csproj -r win-x64
```

### Example 4: Content Builder, Game, and Assets All in Separate Repositories

Maximum separation - each component (builder, assets, game) has its own repository. This is ideal for large organizations with specialized teams.

**In the game repository:**

```yaml
name: Build Game from Multiple Sources

on:
  workflow_dispatch

jobs:
  build:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v5
      
      - name: Checkout content builder action
        uses: actions/checkout@v5
        with:
          repository: MyOrg/monogame-content-builder
          path: .github/actions/build-content
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v5
        with:
          dotnet-version: '9.0.x'
      
      - name: Process content
        uses: ./.github/actions/build-content
        with:
          content-project-path: './ContentPipeline'
          content-builder-repo: 'MyOrg/monogame-content-builder'
          assets-source-path: './GameAssets'
          assets-repo: 'MyOrg/game-assets'
          monogame-platform: 'DesktopGL'
          output-folder: './MyGame/bin/Release/Content'
      
      - name: Build game
        run: dotnet build -c Release MyGame/MyGame.csproj -r win-x64
```

### Example 5: Build and Upload Content Separately

Build content in a dedicated workflow and upload it as an artifact for use by other jobs or workflows. This is useful for content-heavy games where assets are built once and reused across multiple platform builds.

```yaml
name: Build and Distribute Content

on:
  workflow_dispatch
  push:
    paths:
      - 'Content/Assets/**'

jobs:
  build-content:
    runs-on: windows-latest
    strategy:
      matrix:
        platform: [DesktopGL, iOS, Android]
    steps:
      - uses: actions/checkout@v5
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v5
        with:
          dotnet-version: '9.0.x'
      
      - name: Set runtime identifier
        id: set-runtime
        shell: pwsh
        run: |
          $runtime = switch ("${{ matrix.platform }}") {
            "iOS" { "ios-arm64" }
            "Android" { "android-arm64" }
            "DesktopGL" { "win-x64" }
            default { "win-x64" }
          }
          "runtime=$runtime" | Out-File -FilePath $env:GITHUB_OUTPUT -Encoding utf8 -Append
      
      - name: Process content for ${{ matrix.platform }}
        uses: ./.github/actions/build-content
        with:
          content-project-path: './Content'
          assets-source-path: './Content/Assets'
          monogame-platform: ${{ matrix.platform }}
          output-folder: './ContentOutput/${{ matrix.platform }}'
          upload-output: 'true'
      
      # The processed content is now available as:
      # - Artifact: content-output-<run_id>
      # - Location: ./ContentOutput/<platform>/
  
  build-game:
    needs: build-content
    runs-on: windows-latest
    strategy:
      matrix:
        platform: [DesktopGL, iOS, Android]
    steps:
      - uses: actions/checkout@v5
      
      - name: Download content for ${{ matrix.platform }}
        uses: actions/download-artifact@v4
        with:
          name: content-output-${{ github.run_id }}
          path: ./MyGame/bin/Release/Content
      
      - name: Build game
        run: dotnet build -c Release MyGame.${{ matrix.platform }}/MyGame.${{ matrix.platform }}.csproj
```

### Example 6: Using Subfolders for Multi-Configuration Repositories

When your repositories contain multiple builder configurations or asset sets, use the subfolder parameters to target specific directories. This is ideal for organizations managing multiple projects or platform-specific configurations in a single repository.

```yaml
name: Build with Repository Subfolders

on:
  workflow_dispatch:
    inputs:
      platform:
        description: 'Target platform'
        required: true
        type: choice
        options:
          - DesktopGL
          - iOS
          - Android

jobs:
  build:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v5
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v5
        with:
          dotnet-version: '9.0.x'
      
      - name: Process content with subfolders
        uses: ./.github/actions/build-content
        with:
          content-project-path: './ContentBuilder'
          content-builder-repo: 'MyOrg/monogame-content-pipelines'
          content-builder-subfolder: 'builders/${{ github.event.inputs.platform }}'
          assets-source-path: './GameAssets'
          assets-repo: 'MyOrg/game-assets-library'
          assets-subfolder: 'projects/MyGame/common'
          monogame-platform: ${{ github.event.inputs.platform }}
          output-folder: './MyGame/bin/Release/Content'
      
      - name: Build game
        run: dotnet build -c Release MyGame/MyGame.csproj
```

**Example repository structures:**

Content builder repository (`MyOrg/monogame-content-pipelines`):
```
builders/
  DesktopGL/
    Builder.csproj
    Program.cs
  iOS/
    Builder.csproj
    Program.cs
  Android/
    Builder.csproj
    Program.cs
```

Assets repository (`MyOrg/game-assets-library`):
```
projects/
  MyGame/
    common/
      fonts/
      sounds/
    mobile/
      textures/
    desktop/
      textures/
  AnotherGame/
    ...
```

### Example 7: Mixed Configurations with Subfolders

Combine local and external repositories with subfolder targeting for maximum flexibility.

```yaml
name: Complex Build Configuration

on:
  workflow_dispatch

jobs:
  build-desktop:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v5
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v5
        with:
          dotnet-version: '9.0.x'
      
      - name: Build desktop content
        uses: ./.github/actions/build-content
        with:
          # Use local content builder
          content-project-path: './Content/Builder'
          # Pull shared assets from external repo, use desktop-specific subfolder
          assets-source-path: './SharedAssets'
          assets-repo: 'MyOrg/shared-game-assets'
          assets-subfolder: 'desktop-hd'
          monogame-platform: 'DesktopGL'
          output-folder: './MyGame.Desktop/bin/Release/Content'
      
      - name: Build game
        run: dotnet build -c Release MyGame.Desktop/MyGame.Desktop.csproj

  build-mobile:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v5
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v5
        with:
          dotnet-version: '9.0.x'
      
      - name: Build iOS content
        uses: ./.github/actions/build-content
        with:
          # Use external mobile-optimized builder
          content-project-path: './MobileBuilder'
          content-builder-repo: 'MyOrg/mobile-content-builder'
          content-builder-subfolder: 'ios'
          # Use external assets, mobile-optimized subfolder
          assets-source-path: './MobileAssets'
          assets-repo: 'MyOrg/shared-game-assets'
          assets-subfolder: 'mobile-optimized'
          monogame-platform: 'iOS'
          output-folder: './MyGame.iOS/bin/Release/Content'
      
      - name: Build game
        run: dotnet build -c Release MyGame.iOS/MyGame.iOS.csproj
```

## Advanced Usage

### With Custom Arguments

Pass additional arguments to the MonoGame Content Builder:

```yaml
- name: Process content with custom options
  uses: ./.github/actions/build-content
  with:
    content-project-path: './Content'
    assets-source-path: './Content/Assets'
    monogame-platform: 'DesktopGL'
    output-folder: './MyGame/bin/Release/Content'
    additional-args: '--verbose --rebuild --compress'
```

### Multi-Platform Build

Build content for multiple platforms in parallel:

```yaml
jobs:
  build-content:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        include:
          - platform: iOS
            os: macos-latest
            runtime: ios-arm64
          - platform: Android
            os: windows-latest
            runtime: android-arm64
          - platform: DesktopGL
            os: windows-latest
            runtime: win-x64
    steps:
      - uses: actions/checkout@v5
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v5
        with:
          dotnet-version: '9.0.x'
      
      - name: Process content
        uses: ./.github/actions/build-content
        with:
          content-project-path: './Content'
          assets-source-path: './Content/Assets'
          monogame-platform: ${{ matrix.platform }}
          output-folder: './Output/${{ matrix.platform }}'
          upload-output: 'true'
```

## Artifacts

The action automatically uploads two types of artifacts:

### Build Logs (Always)
- **Name**: `content-build-logs-<run_id>`
- **Contains**: 
  - `restore.log` - Content project restore output
  - `build.log` - Content project build output
  - `content-pipeline.log` - Content pipeline execution output
- **Retention**: 30 days
- **When**: Always uploaded (even on failure) for debugging

### Content Output (Optional)
- **Name**: `content-output-<platform>-<run_id>`
- **Contains**: The compiled content files for the specified platform
- **Retention**: 30 days
- **When**: Only when `upload-output: 'true'`

## Troubleshooting

### Content Builder Not Found
**Error**: `Content project path not found` or `No .csproj file found`

**Solution**: 
- Ensure `content-project-path` points to the correct directory
- If using `content-builder-repo`, verify the repository contains a `.csproj` file
- If using `content-builder-subfolder`, ensure the subfolder exists and contains a `.csproj` file
- The action automatically searches recursively for `.csproj` files; if multiple are found, it uses the first one

### Assets Not Found
**Error**: `Assets source path not found` or `Subfolder not found in cloned repository`

**Solution**: 
- Verify `assets-source-path` points to the directory containing your raw assets
- If using `assets-subfolder`, ensure the subfolder exists within the cloned repository
- Check for typos in folder paths (paths are case-sensitive on Linux/macOS runners)

### Clone Failures
**Error**: Git clone fails for `content-builder-repo` or `assets-repo`

**Solution**: 
- Verify the repository exists and is public, or ensure proper authentication
- Check the format is exactly `owner/repo` (no `https://` or `.git`)

### Platform Not Recognized
**Error**: Invalid platform specified

**Solution**: Ensure `monogame-platform` matches one of MonoGame's supported platforms exactly (case-sensitive).

### Content Pipeline Errors
**Error**: Content pipeline fails during processing

**Solution**: 
- Download the `content-build-logs-<run_id>` artifact
- Review `content-pipeline.log` for detailed error messages
- Check asset file formats and content processor settings

## Dependencies

This action requires:
- .NET SDK (version compatible with your MonoGame Content Builder)
- Git (for cloning external repositories)
- MonoGame Content Pipeline tools (included in the content builder project)

## License

This action is provided as-is for use with MonoGame projects.
