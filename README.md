# PrintShareAudit.ps1
# PowerShell script to detect common Windows 11 print-sharing errors
# Run as Administrator

Set-StrictMode -Version Latest
$ErrorActionPreference = "Continue"

$results = @()
$errors = 0

function Add-Result {
    param(
        [string]$Check,
        [string]$Status,
        [string]$Details
    )

    $results += [PSCustomObject]@{
        Check   = $Check
        Status  = $Status
        Details = $Details
    }
}

function Test-ServiceStatus {
    param([string]$ServiceName)

    $svc = Get-Service -Name $ServiceName -ErrorAction SilentlyContinue
    if (-not $svc) {
        Add-Result -Check "Service: $ServiceName" -Status "FAIL" -Details "Service not found."
        return
    }

    if ($svc.Status -eq "Running") {
        Add-Result -Check "Service: $ServiceName" -Status "PASS" -Details "Service is running."
    } else {
        Add-Result -Check "Service: $ServiceName" -Status "FAIL" -Details "Service status: $($svc.Status)"
        $script:errors++
    }
}

function Test-FirewallRule {
    param([string]$RuleName)

    $rule = Get-NetFirewallRule -DisplayGroup "File and Printer Sharing" -ErrorAction SilentlyContinue |
            Where-Object { $_.DisplayName -match $RuleName } |
            Select-Object -First 1

    if ($rule) {
        if ($rule.Enabled -eq "True") {
            Add-Result -Check "Firewall: $RuleName" -Status "PASS" -Details "Rule enabled."
        } else {
            Add-Result -Check "Firewall: $RuleName" -Status "FAIL" -Details "Rule disabled."
            $script:errors++
        }
    } else {
        Add-Result -Check "Firewall: $RuleName" -Status "WARN" -Details "Rule not found."
    }
}

function Get-EventPrintErrors {
    $events = Get-WinEvent -LogName "Microsoft-Windows-PrintService/Operational" -ErrorAction SilentlyContinue |
              Where-Object { $_.LevelDisplayName -match "Error|Warning" } |
              Select-Object -First 20

    if ($events) {
        $msg = ($events | Select-Object -ExpandProperty Message | Out-String).Trim()
        Add-Result -Check "Print Service Logs" -Status "WARN" -Details $msg
    } else {
        Add-Result -Check "Print Service Logs" -Status "PASS" -Details "No recent print service error logs found."
    }
}

Write-Host "=======================================" -ForegroundColor Cyan
Write-Host "Windows 11 Print Sharing Diagnostic Tool" -ForegroundColor Cyan
Write-Host "=======================================" -ForegroundColor Cyan
Write-Host ""

# 1) Basic print services
Test-ServiceStatus -ServiceName "Spooler"
Test-ServiceStatus -ServiceName "LanmanServer"
Test-ServiceStatus -ServiceName "LanmanWorkstation"

# 2) SMB / network
$lanmanServer = Get-Service -Name "LanmanServer" -ErrorAction SilentlyContinue
$lanmanWorkstation = Get-Service -Name "LanmanWorkstation" -ErrorAction SilentlyContinue

if ($lanmanServer -and $lanmanServer.Status -eq "Running" -and
    $lanmanWorkstation -and $lanmanWorkstation.Status -eq "Running") {
    Add-Result -Check "SMB Services" -Status "PASS" -Details "LanmanServer and LanmanWorkstation are running."
} else {
    Add-Result -Check "SMB Services" -Status "FAIL" -Details "One or both SMB services are not running."
    $script:errors++
}

# 3) Firewall
Test-FirewallRule -RuleName "File and Printer Sharing (SMB-In)"
Test-FirewallRule -RuleName "File and Printer Sharing (NB-Session-In)"
Test-FirewallRule -RuleName "File and Printer Sharing (Echo Request - ICMPv4-In)"

# 4) Shared printers
$printers = Get-Printer -ErrorAction SilentlyContinue
if ($printers) {
    $sharedPrinters = $printers | Where-Object { $_.Shared -eq $true }
    if ($sharedPrinters) {
        Add-Result -Check "Shared Printers" -Status "PASS" -Details ($sharedPrinters | ForEach-Object { $_.Name } | Out-String).Trim()
    } else {
        Add-Result -Check "Shared Printers" -Status "WARN" -Details "No shared printers found."
    }
} else {
    Add-Result -Check "Shared Printers" -Status "WARN" -Details "No printers found on this PC."
}

# 5) SMB share check
$smbShares = Get-SmbShare -ErrorAction SilentlyContinue
if ($smbShares) {
    $shareNames = $smbShares | ForEach-Object { $_.Name }
    Add-Result -Check "SMB Shares" -Status "PASS" -Details ($shareNames -join ", ")
} else {
    Add-Result -Check "SMB Shares" -Status "WARN" -Details "No SMB shares detected."
}

# 6) Test local TCP/445 connectivity
$tcp = Test-NetConnection -ComputerName localhost -Port 445 -WarningAction SilentlyContinue
if ($tcp.TcpTestSucceeded) {
    Add-Result -Check "Port 445" -Status "PASS" -Details "Local SMB port 445 is reachable."
} else {
    Add-Result -Check "Port 445" -Status "FAIL" -Details "Local SMB port 445 is not reachable."
    $script:errors++
}

# 7) Event log check
Get-EventPrintErrors

# 8) Display final summary
Write-Host ""
Write-Host "RESULTS:" -ForegroundColor Yellow
$results | Format-Table -AutoSize

$failCount = ($results | Where-Object { $_.Status -eq "FAIL" }).Count
$warnCount = ($results | Where-Object { $_.Status -eq "WARN" }).Count

Write-Host ""
Write-Host "Summary: FAIL=$failCount WARN=$warnCount" -ForegroundColor Yellow

if ($failCount -gt 0) {
    Write-Host ""
    Write-Host "Possible problems found. Check the failed items above." -ForegroundColor Red
    Write-Host "Common fixes:" -ForegroundColor Red
    Write-Host "  - Start the Printer Spooler service" -ForegroundColor Red
    Write-Host "  - Enable 'File and Printer Sharing' in Windows Firewall" -ForegroundColor Red
    Write-Host "  - Ensure SMB services are not disabled" -ForegroundColor Red
    Write-Host "  - Share the printer from Settings > Bluetooth & devices > Printers & scanners" -ForegroundColor Red
    Write-Host "  - Check Event Viewer > Applications and Services Logs > Microsoft > Windows > PrintService" -ForegroundColor Red
} else {
    Write-Host ""
    Write-Host "No critical failures detected in basic print-sharing checks." -ForegroundColor Green
}

Write-Host ""
Write-Host "Tip: To save results to a text file, run:" -ForegroundColor Cyan
Write-Host '  .\PrintShareAudit.ps1 | Tee-Object -FilePath .\PrintShareAudit.txt' -ForegroundColor Cyan
