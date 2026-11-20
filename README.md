# powershell-scripts
Script Powershell per censimenti Jon Veeam e task windows.


# =====================================================================
# INVENTARIO TASK WINDOWS + JOB VEEAM
# EXPORT CSV UNIFICATO
# =====================================================================

$ExportFile = "C:\Temp\Censimento_Task_Veeam.csv"

$Result = @()

# =====================================================================
# TASK WINDOWS
# =====================================================================

Write-Host "Analisi Task Scheduler..."

$Tasks = Get-ScheduledTask

foreach ($Task in $Tasks) {

    try {
        $Info = Get-ScheduledTaskInfo `
            -TaskName $Task.TaskName `
            -TaskPath $Task.TaskPath

        $LastUser = "N/D"

        try {
            $Event = Get-WinEvent `
                -LogName "Microsoft-Windows-TaskScheduler/Operational" `
                -MaxEvents 5000 |
                Where-Object {
                    $_.Id -eq 129 -and
                    $_.Message -like "*$($Task.TaskName)*"
                } |
                Sort-Object TimeCreated -Descending |
                Select-Object -First 1

            if ($Event) {

                if ($Event.Message -match "User:\s+(.+?)\r?\n") {
                    $LastUser = $Matches[1].Trim()
                }
            }
        }
        catch {}

        $Result += [PSCustomObject]@{
            Tipo               = "Windows Task"
            Nome               = $Task.TaskName
            Percorso           = $Task.TaskPath
            Stato              = $Task.State
            EseguitoCome       = $Task.Principal.UserId
            UltimoAvvioDa      = $LastUser
            UltimaEsecuzione   = $Info.LastRunTime
            ProssimaEsecuzione = $Info.NextRunTime
            Risultato          = $Info.LastTaskResult
        }
    }
    catch {
        Write-Warning "Errore task $($Task.TaskName)"
    }
}

# =====================================================================
# LOAD VEEAM MODULE
# =====================================================================

Write-Host "Analisi Job Veeam..."

try {

    if (Get-Module -ListAvailable Veeam.Backup.PowerShell) {
        Import-Module Veeam.Backup.PowerShell -ErrorAction Stop
    }
    else {
        Add-PSSnapin VeeamPSSnapIn -ErrorAction Stop
    }

    $Jobs = Get-VBRJob

    foreach ($Job in $Jobs) {

        try {

            $LastSession = Get-VBRBackupSession |
                Where-Object { $_.JobName -eq $Job.Name } |
                Sort-Object EndTime -Descending |
                Select-Object -First 1

            $StartUser = "N/D"

            try {

                if ($LastSession.CreationTime) {

                    $LogWindowStart = $LastSession.CreationTime.AddMinutes(-2)
                    $LogWindowEnd   = $LastSession.CreationTime.AddMinutes(2)

                    $Events = Get-WinEvent -LogName "Veeam Backup" -ErrorAction SilentlyContinue |
                        Where-Object {
                            $_.TimeCreated -ge $LogWindowStart -and
                            $_.TimeCreated -le $LogWindowEnd -and
                            $_.Message -like "*$($Job.Name)*"
                        } |
                        Select-Object -First 1

                    if ($Events.Message -match "User[: ]+(.+)") {
                        $StartUser = $Matches[1].Trim()
                    }
                }
            }
            catch {}

            $Result += [PSCustomObject]@{
                Tipo               = "Veeam Job"
                Nome               = $Job.Name
                Percorso           = ""
                Stato              = if ($Job.IsScheduleEnabled) { "Schedule Enabled" } else { "Schedule Disabled" }
                EseguitoCome       = ""
                UltimoAvvioDa      = $StartUser
                UltimaEsecuzione   = $LastSession.CreationTime
                ProssimaEsecuzione = ""
                Risultato          = $LastSession.Result
            }
        }
        catch {
            Write-Warning "Errore job Veeam $($Job.Name)"
        }
    }

}
catch {
    Write-Warning "Modulo Veeam non disponibile."
}

# =====================================================================
# EXPORT CSV
# =====================================================================

$Result |
Sort-Object Tipo, Nome |
Export-Csv $ExportFile -NoTypeInformation -Encoding UTF8 -Delimiter ";"

Write-Host ""
Write-Host "Report generato:"
Write-Host $ExportFile -ForegroundColor Green
Write-Host ""
Write-Host "Record esportati: $($Result.Count)"
