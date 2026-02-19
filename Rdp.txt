name: RDP + Tailscale + RustDesk + (Optional) Chrome Remote Desktop (A)

on:
  workflow_dispatch:
    inputs:
      ts_tailnet:         { description: "Tailscale tailnet (e.g. you@gmail.com)", required: true }
      ts_api_key:         { description: "Tailscale API key (device admin, no Bearer)", required: true }
      ts_authkey:         { description: "Tailscale Auth key (reusable or ephemeral)", required: true }
      quick_test:         { description: "Run 5-minute test", type: boolean, default: false }
      runtime_minutes:    { description: "Runtime (max 360; default 355 when not test)", required: false, default: "355" }
      do_purge:           { description: "Purge bullet* devices at start (single instance only)", required: false, default: "true" }
      cycles:             { description: "0=stop after A; N=handoffs left incl this run", required: false, default: "0" }
      rdp_count:          { description: "How many RDP instances (1-10) [ignored if CRD provided]", required: false, default: "1" }
      crd_code:
        description: >-
          Paste raw CRD code (4/0AV...), full PowerShell command, or text/URL containing --code="4/...".
          If provided -> single instance; CRD launches first.
        required: false
        default: ""
      crd_pin:
        description: "CRD PIN (6–12 digits). Used when launching CRD to avoid interactive prompt."
        required: false
        default: "123456"

permissions:
  contents: read
  actions: write

defaults:
  run:
    shell: pwsh

jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.mk.outputs.matrix }}
      multi:  ${{ steps.mk.outputs.multi }}   # expose single vs multi
    steps:
      - id: mk
        run: |
          # Safe literal read (so full commands with &, quotes, ${Env:...} don't break)
          $crd = @'
          ${{ inputs.crd_code }}
          '@.Trim()

          # If CRD provided -> force single; else use rdp_count (1..10)
          if ($crd.Length -gt 0) { $n = 1 } else {
            $n = [int]"${{ inputs.rdp_count }}"; if ($n -lt 1) { $n = 1 }; if ($n -gt 10) { $n = 10 }
          }

          $inc = @(); for ($i=1; $i -le $n; $i++){ $inc += @{ id = $i } }
          $json = @{ include = $inc } | ConvertTo-Json -Compress
          "matrix=$json" | Out-File $env:GITHUB_OUTPUT -Append -Encoding utf8

          # NEW: multi flag
          $multi = if ($n -gt 1) { '1' } else { '0' }
          "multi=$multi" | Out-File $env:GITHUB_OUTPUT -Append -Encoding utf8

  rdp:
    needs: setup
    runs-on: windows-latest
    strategy:
      matrix: ${{ fromJson(needs.setup.outputs.matrix) }}
      max-parallel: 10
    timeout-minutes: 370
    env:
      RDP_USER: Bullettemporary
      RDP_PASS: Bullet@12345

    steps:
      # --- CRD FIRST (prevent code timeout) ---
      - name: Set CRD flag early (safe read)
        run: |
          $crd = @'
          ${{ inputs.crd_code }}
          '@.Trim()
          if ($crd.Length -gt 0) { "CRD_ENABLED=1" | Out-File -Append $env:GITHUB_ENV } else { "CRD_ENABLED=0" | Out-File -Append $env:GITHUB_ENV }
          Write-Host "CRD_ENABLED=$([int]($crd.Length -gt 0))"

      - name: Install Chrome Remote Desktop Host (if missing)  # EXE -> MSI fallback
        if: env.CRD_ENABLED == '1'
        run: |
          $exe = "${Env:PROGRAMFILES(X86)}\Google\Chrome Remote Desktop\CurrentVersion\remoting_start_host.exe"
          if (-not (Test-Path $exe)) {
            Write-Host "CRD host not found, attempting install (EXE -> MSI fallback)..."
            $exePath = "$env:TEMP\chromeremotedesktophost.exe"
            $ok = $false
            try {
              Invoke-WebRequest -Uri "https://dl.google.com/support/remote_assistance/ChromotingHostSetup.exe" -OutFile $exePath -ErrorAction Stop
              Start-Process -FilePath $exePath -ArgumentList "/silent","/install" -Wait
              Remove-Item $exePath -Force
              $ok = $true
            } catch {
              Write-Host "EXE installer failed or missing: $($_.Exception.Message)"
            }
            if (-not $ok) {
              $msi = "$env:TEMP\chromeremotedesktophost.msi"
              try {
                Invoke-WebRequest -Uri "https://dl.google.com/edgedl/chrome-remote-desktop/chromeremotedesktophost.msi" -OutFile $msi -ErrorAction Stop
                Start-Process msiexec.exe -ArgumentList "/i","`"$msi`"","/quiet","/norestart" -Wait
                Remove-Item $msi -Force
              } catch {
