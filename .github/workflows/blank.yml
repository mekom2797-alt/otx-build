name: Build OTX 860

on:
  workflow_dispatch:
    inputs:
      branch:
        description: "otxserv2 (OTX 2.16, 8.60) or otxserv3 (OTX 3.8)"
        default: "otxserv2"
      sdk_url:
        description: "SDK zip URL (only needed for otxserv3, leave empty for otxserv2)"
        required: false
        default: ""

jobs:
  windows:
    runs-on: windows-2022
    steps:
      - name: Clone OTX
        shell: pwsh
        run: |
          git clone --depth 1 --branch "${{ inputs.branch }}" https://github.com/mattyx14/otxserver.git otx
          if ($LASTEXITCODE -ne 0) { exit 1 }
          Get-ChildItem otx -Recurse -Filter *.sln | Select-Object FullName

      - name: Install Boost 1.84
        shell: pwsh
        run: |
          curl.exe -L -o boost.exe "https://sourceforge.net/projects/boost/files/boost-binaries/1.84.0/boost_1_84_0-msvc-14.3-64.exe/download"
          if ($LASTEXITCODE -ne 0) { exit 1 }
          Start-Process -FilePath .\boost.exe -ArgumentList '/VERYSILENT','/SUPPRESSMSGBOXES','/NORESTART','/DIR=C:\local\boost_1_84_0' -Wait
          if (-not (Test-Path C:\local\boost_1_84_0\boost)) { Write-Host "Boost install failed"; exit 1 }

      - name: Install SDK (optional)
        if: ${{ inputs.sdk_url != '' }}
        shell: pwsh
        run: |
          curl.exe -L -o sdk.zip "${{ inputs.sdk_url }}"
          if ($LASTEXITCODE -ne 0) { exit 1 }
          Expand-Archive sdk.zip -DestinationPath C:\sdk -Force
          $root = (Get-ChildItem C:\sdk -Recurse -Filter registerenv.bat | Select-Object -First 1).DirectoryName
          if (-not $root) { Write-Host "registerenv.bat not found"; Get-ChildItem C:\sdk | Select-Object Name; exit 1 }
          Add-Content $env:GITHUB_ENV "SDK-NOBOOST=$root"

      - name: Build
        shell: pwsh
        run: |
          $sln = Get-ChildItem otx -Recurse -Filter TheOTXServer.sln | Select-Object -First 1
          if (-not $sln) { Write-Host "TheOTXServer.sln not found"; exit 1 }
          $msb = & "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" -latest -requires Microsoft.Component.MSBuild -find MSBuild\**\Bin\MSBuild.exe | Select-Object -First 1
          & $msb $sln.FullName /m /p:Configuration=Release /p:Platform=x64 /p:PlatformToolset=v143 /verbosity:minimal 2>&1 | Tee-Object -FilePath build.log
          if ($LASTEXITCODE -ne 0) { exit 1 }

      - name: Show errors
        if: failure()
        shell: pwsh
        run: |
          if (Test-Path build.log) {
            $s = Select-String -Path build.log -Pattern 'error C\d+|error LNK\d+|error MSB\d+|fatal error' | Select-Object -Last 40 | ForEach-Object { $_.Line }
            $s
            $s | Out-File $env:GITHUB_STEP_SUMMARY -Encoding utf8
          } else { "build.log not found" }

      - name: Collect
        shell: pwsh
        run: |
          New-Item -ItemType Directory -Force out | Out-Null
          Get-ChildItem otx -Recurse -Include *.exe |
            Where-Object { $_.FullName -match 'x64' -and $_.FullName -match 'Release' } |
            Copy-Item -Destination out
          Get-ChildItem otx -Recurse -Include *.dll |
            Where-Object { $_.FullName -match 'x64' -and $_.FullName -match 'Release' } |
            Copy-Item -Destination out -ErrorAction SilentlyContinue
          foreach ($d in 'data','config.lua','config.lua.dist','schema.sql','schemas') {
            $p = Join-Path otx $d
            if (Test-Path $p) { Copy-Item $p out -Recurse }
          }
          Get-ChildItem out

      - uses: actions/upload-artifact@v4
        with:
          name: otx-windows
          path: out

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: otx-log
          path: build.log
          if-no-files-found: ignore
