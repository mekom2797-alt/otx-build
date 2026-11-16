name: Build OTX 860

on:
  workflow_dispatch:
    inputs:
      branch:
        description: "otxserv2 (OTX 2.16, 8.60) or otxserv3 (OTX 3.8)"
        default: "otxserv2"
      sdk_url:
        description: "SDK zip URL (only for otxserv3, leave empty for otxserv2)"
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

      - name: Inspect project dependencies
        shell: pwsh
        run: |
          $proj = Get-ChildItem otx -Recurse -Filter TheOTXServer.vcxproj | Select-Object -First 1
          Write-Host "Project: $($proj.FullName)"
          $t = Get-Content $proj.FullName -Raw
          foreach ($tag in 'AdditionalIncludeDirectories','AdditionalLibraryDirectories','AdditionalDependencies','PreprocessorDefinitions','PlatformToolset') {
            Write-Host "=== $tag ==="
            [regex]::Matches($t, "<$tag>(.*?)</$tag>") | ForEach-Object { $_.Groups[1].Value } | Sort-Object -Unique
          }
          Write-Host "=== platforms in sln ==="
          Select-String -Path (Get-ChildItem otx -Recurse -Filter TheOTXServer.sln | Select-Object -First 1).FullName -Pattern 'Release\|' | Select-Object -First 10 | ForEach-Object { $_.Line.Trim() }

      - name: Build with auto-fix
        id: build
        shell: pwsh
        run: |
          $ErrorActionPreference = 'Continue'
          $vc        = $env:VCPKG_INSTALLATION_ROOT
          $inst      = "$vc\installed\x64-windows"
          $boostRoot = 'C:\local\boost_1_84_0'
          $boostLib  = "$boostRoot\lib64-msvc-14.3"
          $shim      = Join-Path $PWD 'libshim'
          New-Item -ItemType Directory -Force $shim | Out-Null
          $env:BOOST_ROOT = $boostRoot

          $extraInc = New-Object System.Collections.Generic.List[string]
          $extraLib = New-Object System.Collections.Generic.List[string]
          $defines  = New-Object System.Collections.Generic.List[string]
          foreach ($d in '_HAS_AUTO_PTR_ETC=1','_CRT_SECURE_NO_WARNINGS','_WINSOCK_DEPRECATED_NO_WARNINGS','_SILENCE_ALL_CXX17_DEPRECATION_WARNINGS') { $defines.Add($d) }
          $extraInc.Add($boostRoot)
          $extraLib.Add($shim); $extraLib.Add("$inst\lib")
          if (Test-Path $boostLib) { $extraLib.Add($boostLib) }
          $tried = @{}

          function Install-Port([string]$port) {
            if ($tried.ContainsKey($port)) { return $false }
            $tried[$port] = $true
            Write-Host ">>> vcpkg install $port"
            & "$vc\vcpkg.exe" install "${port}:x64-windows" --disable-metrics 2>&1 | Tee-Object -FilePath "vcpkg-$port.log" | Select-Object -Last 3
            return ($LASTEXITCODE -eq 0)
          }

          function Ports-ForHeader([string]$h) {
            $h = $h.ToLower()
            switch -regex ($h) {
              '^libxml/'                         { return @('libxml2') }
              '^(lua|lauxlib|lualib|luaconf|luajit)' { return @('luajit') }
              '^(mysql|errmsg|mysqld_error)'     { return @('libmariadb') }
              '^gmp'                             { return @('gmp','mpir') }
              '^openssl/'                        { return @('openssl') }
              '^zlib'                            { return @('zlib') }
              '^sqlite3'                         { return @('sqlite3') }
              '^pugi'                            { return @('pugixml') }
              '^crypto'                          { return @('cryptopp') }
              '^curl/'                           { return @('curl') }
              '^boost/'                          { return @() }
              default { return @(([IO.Path]::GetFileNameWithoutExtension($h)) -replace '[^a-z0-9]','') }
            }
          }

          function Ports-ForLib([string]$l) {
            $l = ($l.ToLower() -replace '\.lib$','')
            switch -regex ($l) {
              'xml'               { return @('libxml2') }
              'lua'               { return @('luajit') }
              'mysql|mariadb'     { return @('libmariadb') }
              'gmp|mpir'          { return @('gmp','mpir') }
              'ssl|eay|crypto'    { return @('openssl') }
              'zlib|^z$'          { return @('zlib') }
              'sqlite'            { return @('sqlite3') }
              default             { return @(($l -replace '^lib','')) }
            }
          }

          function Find-IncludeRoot([string]$h) {
            $rel = $h -replace '/','\'
            if (Test-Path (Join-Path $boostRoot $rel)) { return $boostRoot }
            $name = Split-Path $rel -Leaf
            $c = Get-ChildItem "$inst\include" -Recurse -Filter $name -File -ErrorAction SilentlyContinue |
                 Where-Object { $_.FullName.EndsWith($rel, [StringComparison]::OrdinalIgnoreCase) } | Select-Object -First 1
            if ($c) { return $c.FullName.Substring(0, $c.FullName.Length - $rel.Length).TrimEnd('\') }
            return $null
          }

          $alias = @{ 'mysql'=@('mariadb','mysql'); 'gmp'=@('mpir','gmp'); 'ssleay'=@('libssl','ssl'); 'libeay'=@('libcrypto','crypto'); 'ssl'=@('libssl'); 'crypto'=@('libcrypto') }
          function Norm([string]$n) { return (($n.ToLower() -replace '\.lib$','' -replace '^lib','') -replace '[^a-z]','') }

          function Find-LibMatch([string]$wanted) {
            $w = Norm $wanted
            $names = @($w); if ($alias.ContainsKey($w)) { $names += $alias[$w] }
            $pool = @()
            $pool += Get-ChildItem "$inst\lib" -Filter *.lib -File -ErrorAction SilentlyContinue
            if (Test-Path $boostLib) { $pool += Get-ChildItem $boostLib -Filter *.lib -File -ErrorAction SilentlyContinue }
            foreach ($n in $names) {
              $nn = Norm $n
              $hit = $pool | Where-Object { (Norm $_.Name) -eq $nn } | Select-Object -First 1
              if ($hit) { return $hit.FullName }
            }
            foreach ($n in $names) {
              $nn = Norm $n
              if ($nn.Length -lt 3) { continue }
              $hit = $pool | Where-Object { $cn = Norm $_.Name; $cn.Length -ge 3 -and ($cn.Contains($nn) -or $nn.Contains($cn)) } | Select-Object -First 1
              if ($hit) { return $hit.FullName }
            }
            return $null
          }

          function Write-Targets {
            $inc = ($extraInc -join ';'); $lib = ($extraLib -join ';'); $defs = ($defines -join ';')
            $xml = @"
          <Project>
            <PropertyGroup>
              <IncludePath>`$(IncludePath);$inc</IncludePath>
              <LibraryPath>`$(LibraryPath);$lib</LibraryPath>
            </PropertyGroup>
            <ItemDefinitionGroup>
              <ClCompile>
                <PreprocessorDefinitions>$defs;%(PreprocessorDefinitions)</PreprocessorDefinitions>
                <TreatWarningAsError>false</TreatWarningAsError>
                <SDLCheck>false</SDLCheck>
                <AdditionalOptions>/FS /wd4996 /wd4244 /wd4267 %(AdditionalOptions)</AdditionalOptions>
              </ClCompile>
            </ItemDefinitionGroup>
          </Project>
          "@
            Set-Content -Path "$PWD\otx\Directory.Build.targets" -Value $xml -Encoding UTF8
          }

          $sln = (Get-ChildItem otx -Recurse -Filter TheOTXServer.sln | Select-Object -First 1).FullName
          if (-not $sln) { Write-Host "TheOTXServer.sln not found"; exit 1 }
          $msb = & "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" -latest -requires Microsoft.Component.MSBuild -find MSBuild\**\Bin\MSBuild.exe | Select-Object -First 1

          $seen = @{}
          $ok = $false
          foreach ($i in 1..12) {
            Write-Targets
            Write-Host "=============== build attempt $i ==============="
            & $msb $sln /m /nologo /p:Configuration=Release /p:Platform=x64 /p:PlatformToolset=v143 /verbosity:minimal 2>&1 | Tee-Object -FilePath build.log
            if ($LASTEXITCODE -eq 0) { $ok = $true; break }

            $log = Get-Content build.log -Raw
            $changed = $false

            foreach ($m in [regex]::Matches($log, "Cannot open include file: '([^']+)'")) {
              $h = $m.Groups[1].Value
              if ($seen["H:$h"]) { continue }; $seen["H:$h"] = $true
              $root = Find-IncludeRoot $h
              if (-not $root) {
                foreach ($p in (Ports-ForHeader $h)) {
                  if (Install-Port $p) { $root = Find-IncludeRoot $h; if ($root) { break } }
                }
              }
              if ($root) {
                if (-not $extraInc.Contains($root)) { $extraInc.Add($root); $changed = $true; Write-Host "FIX include: $h -> $root" }
              } else { Write-Host "UNRESOLVED header: $h" }
            }

            foreach ($m in [regex]::Matches($log, "cannot open (?:input )?file '([^']+)'")) {
              $f = Split-Path $m.Groups[1].Value -Leaf
              if ($seen["L:$f"] -or $f -notmatch '\.lib$') { continue }; $seen["L:$f"] = $true
              $hit = Find-LibMatch $f
              if (-not $hit) {
                foreach ($p in (Ports-ForLib $f)) {
                  if (Install-Port $p) { $hit = Find-LibMatch $f; if ($hit) { break } }
                }
              }
              if ($hit) { Copy-Item $hit (Join-Path $shim $f) -Force; $changed = $true; Write-Host "FIX lib: $f <- $hit" }
              else { Write-Host "UNRESOLVED lib: $f" }
            }

            if (($log -match 'C2589|C2059[^\r\n]*\b(min|max)\b') -and -not $defines.Contains('NOMINMAX')) {
              $defines.Add('NOMINMAX'); $changed = $true; Write-Host "FIX: NOMINMAX"
            }
            if ($log -match 'MSB4126|MSB4057[^\r\n]*x64') { Write-Host "Project has no x64 configuration"; break }
            if (-not $changed) { Write-Host "No known fix matched, stopping."; break }
          }
          if (-not $ok) { exit 1 }

      - name: Show errors
        if: failure()
        shell: pwsh
        run: |
          $s = @()
          if (Test-Path build.log) {
            $s += "=== unique errors (last 60) ==="
            $s += Select-String -Path build.log -Pattern 'error C\d+|error LNK\d+|error MSB\d+|fatal error|warning LNK4272' |
                  ForEach-Object { $_.Line.Trim() } | Select-Object -Unique | Select-Object -Last 60
          } else { $s += "build.log not found" }
          $s
          $s | Out-File $env:GITHUB_STEP_SUMMARY -Encoding utf8

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
          Get-ChildItem "$env:VCPKG_INSTALLATION_ROOT\installed\x64-windows\bin" -Filter *.dll -ErrorAction SilentlyContinue |
            Copy-Item -Destination out -ErrorAction SilentlyContinue
          Get-ChildItem 'C:\local\boost_1_84_0\lib64-msvc-14.3' -Filter *.dll -ErrorAction SilentlyContinue |
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
          path: |
            build.log
            vcpkg-*.log
            otx/Directory.Build.targets
          if-no-files-found: ignore
