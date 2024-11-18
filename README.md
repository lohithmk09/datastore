# datastore
Code for Automated Alert System for Server Data Store Capacity Management
# Define global parameters
param (
    [string[]]$Servers = @("localhost", "Server1", "Server2"), # Replace with server names or IPs
    [int]$Threshold = 20, # Set the disk space threshold percentage
    [string]$SMTPServer = "smtp.yourdomain.com", # Replace with your SMTP server
    [string]$FromEmail = "alert@yourdomain.com", # Replace with sender's email address
    [string]$ToEmail = "user@yourdomain.com"     # Replace with recipient's email address
)

# Import necessary modules
Import-Module -Name "Microsoft.PowerShell.Management" -ErrorAction SilentlyContinue

# Function to get disk space details
function Get-DiskSpace {
    param (
        [string]$Server
    )
    try {
        $disks = Get-WmiObject -Class Win32_LogicalDisk -ComputerName $Server -Filter "DriveType=3" -ErrorAction Stop
        $diskDetails = @()
        foreach ($disk in $disks) {
            $FreeSpaceGB = [math]::Round($disk.FreeSpace / 1GB, 2)
            $TotalSpaceGB = [math]::Round($disk.Size / 1GB, 2)
            $UsedSpacePercent = [math]::Round(((($disk.Size - $disk.FreeSpace) / $disk.Size) * 100), 2)
            $diskDetails += [PSCustomObject]@{
                Server          = $Server
                Drive           = $disk.DeviceID
                FreeSpaceGB     = $FreeSpaceGB
                TotalSpaceGB    = $TotalSpaceGB
                UsedSpacePercent = $UsedSpacePercent
            }
        }
        return $diskDetails
    } catch {
        Write-Error "Failed to retrieve disk space for server $Server: $_"
    }
}

# Function to send email alert
function Send-EmailAlert {
    param (
        [string]$Subject,
        [string]$Body
    )
    try {
        $message = @{
            To       = $ToEmail
            From     = $FromEmail
            Subject  = $Subject
            Body     = $Body
            SmtpServer = $SMTPServer
            BodyAsHtml = $true
        }
        Send-MailMessage @message
        Write-Host "Email alert sent successfully to $ToEmail" -ForegroundColor Green
    } catch {
        Write-Error "Failed to send email: $_"
    }
}

# Main script logic
Write-Host "Starting disk space monitoring..." -ForegroundColor Cyan

foreach ($server in $Servers) {
    $diskDetails = Get-DiskSpace -Server $server
    if ($diskDetails) {
        foreach ($disk in $diskDetails) {
            if ($disk.UsedSpacePercent -gt (100 - $Threshold)) {
                # Compose alert message
                $subject = "Low Disk Space Alert: $($disk.Server) - $($disk.Drive)"
                $body = @"
<h2>Disk Space Alert</h2>
<p><strong>Server:</strong> $($disk.Server)</p>
<p><strong>Drive:</strong> $($disk.Drive)</p>
<p><strong>Total Space:</strong> $($disk.TotalSpaceGB) GB</p>
<p><strong>Free Space:</strong> $($disk.FreeSpaceGB) GB</p>
<p><strong>Used Space:</strong> $($disk.UsedSpacePercent)%</p>
<p><strong>Threshold:</strong> $Threshold%</p>
<p>Please take immediate action to free up space.</p>
"@

                # Send email alert
                Send-EmailAlert -Subject $subject -Body $body
            } else {
                Write-Host "Disk space on $($disk.Server):$($disk.Drive) is sufficient." -ForegroundColor Green
            }
        }
    }
}

Write-Host "Disk space monitoring completed." -ForegroundColor Cyan
