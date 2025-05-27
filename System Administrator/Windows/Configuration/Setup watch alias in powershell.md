# Setup `watch` alias command in `Powershell`

`watch` command is very helpful to monitor can using with any command in linux, Unfortunately in windows we doesn't have this command. This guide will try to create custom function to use `watch` in `powershell`

## Create custom function

you can edit `$PROFILE` to add `alias` or `custom script` in powershell eg:

```powershell
notepad $PROFILE
```

add you custom function:

```powershell
function watch {
    param (
        [int]$interval,
        [Parameter(ValueFromRemainingArguments = $true)]
        [string[]]$command
    )

    if (-not $command -or $command.Count -eq 0) {
        Write-Host "❌ You must specify a command to watch. Example: watch 5 kubectl get pods"
        return
    }

    while ($true) {
        cls
        try {
            Invoke-Expression ($command -join " ")
        } catch {
            Write-Host "❌ Error running command: $_"
        }
        Start-Sleep -Seconds $interval
    }
}
```

and reload profile

```powershell
. $PROFILE
```

## Let's try

try to use new custom script like:

```powershell
watch 5 kubectl get pods
```

let fun, If you have more idea and improvement please let me know to improve it.
