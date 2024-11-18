# datastore
Code for Automated Alert System for Server Data Store Capacity Management
# Define global parameters

# Global Parameters

# List of servers to monitor
$Global:Servers = @("localhost", "Server1", "Server2") # Replace with actual server names/IPs

# Disk space alert threshold (percentage)
$Global:Threshold = 20 # Alert if free space is below 20%

# SMTP server settings
$Global:SMTPServer = "smtp.yourdomain.com" # Replace with your SMTP server
$Global:FromEmail = "alert@yourdomain.com" # Sender email address
$Global:ToEmail = "user@yourdomain.com" # Recipient email address (comma-separated for multiple)

# Log file path
$Global:LogFilePath = "$PSScriptRoot\DiskAlertLog.txt"

# Logging format
$Global:DateFormat = "yyyy-MM-dd HH:mm:ss"

# Import necessary modules
Import-Module -Name "Microsoft.PowerShell.Management" -ErrorAction SilentlyContinue

# Function to get disk space details
# Function to get disk space details
function Get-DiskSpaceDetails {
    param (
        [string]$ServerName
    )

    try {
        # Get logical disk information from the specified server
        $diskInfo = Get-WmiObject -Class Win32_LogicalDisk -ComputerName $ServerName -ErrorAction Stop |
            Where-Object { $_.DriveType -eq 3 } # 3 = Local Disk

        # Create a structured result for each disk
        $diskDetails = foreach ($disk in $diskInfo) {
            [PSCustomObject]@{
                ServerName    = $ServerName
                DriveLetter   = $disk.DeviceID
                FreeSpaceGB   = [math]::Round($disk.FreeSpace / 1GB, 2)
                TotalSpaceGB  = [math]::Round($disk.Size / 1GB, 2)
                FreeSpacePerc = [math]::Round(($disk.FreeSpace / $disk.Size) * 100, 2)
            }
        }

        return $diskDetails
    } catch {
        Write-Error "Failed to retrieve disk space details for server: $ServerName. Error: $_"
        return $null
    }
}


# Function to send email alert
# Function to send email alert
function Send-EmailAlert {
    param (
        [string]$ServerName,
        [string]$DriveLetter,
        [float]$FreeSpaceGB,
        [float]$FreeSpacePerc,
        [string]$RecipientEmail,
        [string]$SenderEmail,
        [string]$SMTPServer,
        [int]$SMTPPort = 587,
        [string]$SMTPUser,
        [string]$SMTPPassword
    )

    try {
        # Construct the email subject and body
        $subject = "ALERT: Low Disk Space on $ServerName ($DriveLetter)"
        $body = @"
Dear User,

This is an automated notification regarding low disk space on the following server:

Server Name: $ServerName
Drive Letter: $DriveLetter
Free Space: $FreeSpaceGB GB
Free Space Percentage: $FreeSpacePerc%

Please take immediate action to free up space or expand the disk to avoid potential system issues.

Best regards,
Automated Disk Space Monitoring System
"@

        # Create the email message
        $emailMessage = @{
            From       = $SenderEmail
            To         = $RecipientEmail
            Subject    = $subject
            Body       = $body
            SmtpServer = $SMTPServer
            Port       = $SMTPPort
            UseSsl     = $true
            Credential = New-Object System.Management.Automation.PSCredential ($SMTPUser, (ConvertTo-SecureString $SMTPPassword -AsPlainText -Force))
        }

        # Send the email
        Send-MailMessage @emailMessage

        Write-Host "Email alert sent successfully to $RecipientEmail for $ServerName ($DriveLetter)."
    } catch {
        Write-Error "Failed to send email alert. Error: $_"
    }
}

Parameters:

$ServerName: The name of the server with low disk space.
$DriveLetter: The drive experiencing low disk space (e.g., C:).
$FreeSpaceGB: Amount of free space in GB.
$FreeSpacePerc: Percentage of free space remaining.
$RecipientEmail: Email address of the alert recipient.
$SenderEmail: Email address of the sender.
$SMTPServer: The SMTP server used for sending the email.
$SMTPPort: The SMTP server port (default: 587 for secure email).
$SMTPUser: SMTP username for authentication.
$SMTPPassword: SMTP password for authentication.
Logic:

Constructs an alert email with the disk details in the body.
Uses the Send-MailMessage cmdlet to send the email.
Error Handling:

Captures errors during email sending and logs them.
Email Formatting:

The email contains server name, drive letter, free space, and a brief message for action.

# Main script logic
Write-Host "Starting disk space monitoring..." -ForegroundColor Cyan

# Main script logic
# Define thresholds and other global parameters
$LowDiskSpaceThresholdGB = 10 # Alert if free space is below 10 GB
$LowDiskSpaceThresholdPerc = 10 # Alert if free space is below 10%
$RecipientEmail = "admin@example.com"
$SenderEmail = "monitor@example.com"
$SMTPServer = "smtp.example.com"
$SMTPPort = 587
$SMTPUser = "smtp_user"
$SMTPPassword = "smtp_password"

# Get all servers to monitor
$Servers = @("Server1", "Server2", "Server3") # Add server names or IPs here

# Iterate through each server and check disk space
foreach ($Server in $Servers) {
    try {
        # Get the disk space details
        $Drives = Get-DiskSpace -ServerName $Server

        foreach ($Drive in $Drives) {
            $DriveLetter = $Drive.DriveLetter
            $FreeSpaceGB = $Drive.FreeSpaceGB
            $FreeSpacePerc = $Drive.FreeSpacePerc

            # Check if the free space is below the defined thresholds
            if ($FreeSpaceGB -lt $LowDiskSpaceThresholdGB -or $FreeSpacePerc -lt $LowDiskSpaceThresholdPerc) {
                Write-Host "Low disk space detected on $Server ($DriveLetter): $FreeSpaceGB GB remaining ($FreeSpacePerc%). Sending alert."

                # Send an email alert
                Send-EmailAlert -ServerName $Server `
                    -DriveLetter $DriveLetter `
                    -FreeSpaceGB $FreeSpaceGB `
                    -FreeSpacePerc $FreeSpacePerc `
                    -RecipientEmail $RecipientEmail `
                    -SenderEmail $SenderEmail `
                    -SMTPServer $SMTPServer `
                    -SMTPPort $SMTPPort `
                    -SMTPUser $SMTPUser `
                    -SMTPPassword $SMTPPassword
            } else {
                Write-Host "Disk space on $Server ($DriveLetter) is sufficient: $FreeSpaceGB GB remaining ($FreeSpacePerc%)."
            }
        }
    } catch {
        Write-Error "Failed to monitor disk space on $Server. Error: $_"
    }
}

Explanation:
Thresholds:

$LowDiskSpaceThresholdGB: The minimum free space (in GB) before sending an alert.
$LowDiskSpaceThresholdPerc: The minimum free space percentage before sending an alert.
Server List:

$Servers: An array of server names or IPs to monitor.
Disk Space Check:

Uses the Get-DiskSpace function to retrieve disk information for each server.
Loops through each drive on the server and checks against the thresholds.
Email Alerts:

If a disk meets the low space criteria, the script calls Send-EmailAlert to notify the user.
Error Handling:

Logs errors if a server or drive check fails.
Logging:

Outputs status messages to the console to indicate the script's progress and actions.
