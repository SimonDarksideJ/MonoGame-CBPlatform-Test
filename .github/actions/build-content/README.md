# Build MonoGame Content Action

A reusable composite GitHub Action for building MonoGame content using the MonoGame Content Pipeline Builder. This action provides flexibility to work with content builders and assets from local or external repositories.

## Features

- Build MonoGame content for any platform (iOS, Android, DesktopGL, Windows, etc.)
- Support for local or external content builder repositories
- Support for local or external asset repositories
- Automatic temp folder management
- Comprehensive logging with artifact uploads
- Optional content output archiving and upload
- Custom argument support for advanced scenarios

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `content-project-path` | Yes | `./CBPlatformTest/Content` | Path to the content project folder (relative to repo root). This is where the MonoGame Content Builder console application resides. |
| `content-builder-repo` | No | `''` | Optional GitHub repository for the content builder in `owner/repo` format. If provided, the repository will be cloned to the `content-project-path` location. Leave empty to use a local content builder. |
| `assets-source-path` | Yes | `./Assets` | Path to the assets source folder (relative to repo root or content project). This contains your game's raw assets (images, sounds, fonts, etc.). |
| `assets-repo` | No | `''` | Optional GitHub repository for assets in `owner/repo` format. If provided, the repository will be cloned to the `assets-source-path` location. Leave empty to use local assets. |
| `monogame-platform` | Yes | - | MonoGame platform target. Valid values: `iOS`, `Android`, `DesktopGL`, `Windows`, `WindowsStoreApp`, `MacOSX`, `Linux`, `PlayStation4`, `XboxOne`, `Switch`, etc. |
| `output-folder` | Yes | - | Output folder for processed content (relative to repo root). The compiled content will be placed here, ready to be included in your game build. |
| `runtime-identifier` | **Yes** | - | Runtime identifier for the content builder (e.g., `win-x64`, `linux-x64`, `osx-arm64`, `ios-arm64`, `android-arm64`). Required to avoid assets file conflicts when building for specific platforms. Should match the target platform's runtime. |
| `additional-args` | No | `''` | Additional arguments to pass to the content builder CLI (e.g., `--verbose` or `--rebuild`). Arguments should be space-separated. |
| `upload-output` | No | `false` | Whether to archive and upload the content output as a GitHub artifact. Set to `true` to enable. Useful for debugging or distributing pre-built content. |

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
          runtime-identifier: 'win-x64'
      
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
          runtime-identifier: 'win-x64'
      
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
          runtime-identifier: 'win-x64'
      
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
          runtime-identifier: 'win-x64'
      
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
          runtime-identifier: ${{ steps.set-runtime.outputs.runtime }}
      
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
          runtime-identifier: ${{ matrix.runtime }}
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
- **Name**: `content-output-<run_id>`
- **Contains**: `content-output.zip` - The compiled content files
- **Retention**: 30 days
- **When**: Only when `upload-output: 'true'`

## Troubleshooting

### Content Builder Not Found
**Error**: `Content project path not found`

**Solution**: Ensure `content-project-path` points to the correct directory containing the `.csproj` file.

### Assets Not Found
**Error**: `Assets source path not found`

**Solution**: Verify `assets-source-path` points to the directory containing your raw assets.

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

### Assets File Conflicts (NETSDK1047)
**Error**: `NETSDK1047: Assets file ... doesn't have a target for 'net9.0/...'`

**Solution**: Specify the `runtime-identifier` parameter matching your target platform:
- iOS: `ios-arm64`
- Android: `android-arm64`
- Windows: `win-x64`
- Linux: `linux-x64`
- macOS: `osx-arm64`

This ensures the content builder is restored with the correct runtime target, preventing downstream build errors.

## Dependencies

This action requires:
- .NET SDK (version compatible with your MonoGame Content Builder)
- Git (for cloning external repositories)
- MonoGame Content Pipeline tools (included in the content builder project)

## License

This action is provided as-is for use with MonoGame projects.
