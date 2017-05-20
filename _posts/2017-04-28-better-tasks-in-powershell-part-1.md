---
layout: post
title: "Better Tasks in PowerShell (Part 1): A New Start"
description: "Out of the box work with Task scheduler and powershell"
modified: 2017-05-01
tags: [powershell task]
---
PowerShell allows SysAdmins to write simple tools which can be distributed to lower level support or even users, which in turn give more time to SysAdmins to work on more important business (watching YouTube, trolling #powershell on slack...). But, in some case, it's possible to totally remove the middle-man and write a script that will run periodically, without any user input. This serie of Post will focus on the differences between writing code for users and for full automation. If you want more information about the best way to write user-friendly code, there are already a lot of ressource on that all around the web.

This first post describes a sample task that notify It admins when AD accounts will expire and how to set it using Powershell. The code may look rough, but it will be refined after each post. Let's dig in!

###### SendAccountExpiryReport.ps1
{% highlight PowerShell %}
$ExpiryLimitInDays = 14
$AccountExpiryReportMailSettings = @{
    Subject    = "[REPORT] account expiring in $ExpiryLimitInDays days"
    From       = 'IT-robot@company.com'
    To         = 'IT-team@company.com'
    Encoding   = 'utf-8'
    SmtpServer = 'ns1.company.com'
}

$expiryDateMin = (Get-Date).AddDays($ExpiryLimitInDays)
$expiryDateMax = (Get-Date).AddDays($ExpiryLimitInDays+1)

$allLimitedAccounts = Get-ADUser -Filter "AccountExpirationdate -like '*'" -Properties AccountExpirationdate | Where-Object { $_.AccountExpirationdate -gt $expiryDateMin -and $_.AccountExpirationdate -lt $expiryDateMax }

$AccountExpiryReportMailSettings['Body'] = if ($allLimitedAccounts) {
    ($allLimitedAccounts | Select-Object @{n='Long Account';e={$_.samaccountname}},@{n='Account Expiration Date';e={$_.AccountExpirationdate}} | ConvertTo-Html) -join "`r`n"
} else {
    "nothing today."
}

Send-MailMessage @AccountExpiryReportMailSettings -BodyAsHtml
{% endhighlight %}

Let's say the file is under ```C:\Tasks\SendAccountExpiryReport```. Even if it's not as straightforward as it was in VBS/batch, adding a Scheduled Task to run a PowerShell script is still relatively simple. 

###### Register-SendAccountExpiryReport.ps1
{% highlight PowerShell %}
$scriptPath = "C:\Batches\Tasks\SendAccountExpiryReport\SendAccountExpiryReport.ps1"
$ActionArgument = "-noProfile -ExecutionPolicy Bypass -File `"$scriptPath`""
$action = New-ScheduledTaskAction -Execute 'Powershell.exe' -Argument $ActionArgument
$trigger =  New-ScheduledTaskTrigger -DaysInterval 1 -Daily -At (Get-Date)
$settings = New-ScheduledTaskSettingsSet -Compatibility Win7 -Hidden -AllowStartIfOnBatteries
Register-ScheduledTask -Action $action -Trigger $trigger -TaskName "MyTask-SendAccountExpiryReport" -Settings $settings
{% endhighlight %}

Compared to a batch, you need to execute powershell.exe and pass the script path as parameter. We also use ```-noProfile``` to be sure to "isolate" the script from the environment, it makes for easier integration in the long run. In a similar manner, we bypass the current ```ExecutionPolicy```, so we don't have to set it session/computer wide.

That's enough code to start to be dangerous, since we don't do any test, we don't have log or run it unmonitored, but it's also a short way to get value from PowerShell. Once the script is running, it will run until the cold death of the Universe (or until someone shut down the VM where it's running...).

Next time, we generate documentation for the task... but in the laziest way possible. And we do it soon!