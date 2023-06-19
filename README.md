# File-Integrity-Monitor

## Overview 

This project will demonstrate a Powershell script on file integrity. Integrity is part of the CIA triad acryonym. The CIA triad is a common model that forms the basis for the development of security systems. Integrity focuses on guarding against improper information modification or destruction. A file integrity monitor (FIM) detects file changes that could be an indicator of compromise for a cyberattack.

## Powershell Script breakdown

The following is the Powershell Script for our FIM. The script can be found as QFIM.ps1 in this repository. 

It has two primary functionalities based on the user's input:

Generate a baseline of files in the .\Files directory, or monitor changes in those files, based on the generated baseline. Here's a more detailed step-by-step walkthrough of the script:

Function Calculate-File-Hash: This function takes a file path as an argument and returns the SHA512 hash of the file. Hashing is a common way to verify the integrity of data, as any change in the data will result in a different hash.

<b>To expand on the above function: Calculate-File-Hash($filepath): This function takes a file path as an argument and calculates its SHA512 hash using the built-in Get-FileHash cmdlet in PowerShell. Hashing is a common way to verify the integrity of data, as any change in the data will result in a different hash. After calculating the hash of the file, the function then returns this hash. The function is useful for identifying changes in a file.</b>



Function Erase-Baseline-If-Already-Exists: This function checks if a file called "baseline.txt" exists in the current directory and deletes it if it does.

<b>To expand on the above function: Erase-Baseline-If-Already-Exists(): This function checks if a file named "baseline.txt" already exists in the current directory using the Test-Path cmdlet. If the file exists (Test-Path returns True), it removes the file using the Remove-Item cmdlet. This function is useful for resetting the baseline file that stores the original hashes of the files in the monitored directory. Resetting this baseline is necessary when we want to create a new baseline of file hashes.</b>


The script then presents the user with two options:

Option A: Collect new Baseline: If the user selects this option, the script first calls the Erase-Baseline-If-Already-Exists function to remove any existing baseline file. Then it collects all files in the .\Files directory, calculates the hash for each file using the Calculate-File-Hash function, and appends each file's path and hash to the "baseline.txt" file.

Option B: Begin monitoring files with saved Baseline: If the user selects this option, the script starts an infinite loop that performs the following tasks every second:

Loads the existing file paths and hashes from the "baseline.txt" file into a dictionary for quick lookup.

For each file in the .\Files directory, it calculates the file's hash and checks whether the file path exists in the baseline. If the path does not exist in the baseline, the script assumes the file is newly created and prints a message stating this.

If the file path exists in the baseline, the script checks whether the calculated hash matches the hash from the baseline. If the hashes don't match, it assumes the file has been modified and prints a message stating this.

The script also checks if each file from the baseline still exists in the .\Files directory. If a file from the baseline does not exist in the directory, the script assumes the file has been deleted and prints a message stating this.

## Powershell Script

﻿﻿Function Calculate-File-Hash($filepath) {
    $filehash = Get-FileHash -Path $filepath -Algorithm SHA512
    return $filehash
}
Function Erase-Baseline-If-Already-Exists() {
    $baselineExists = Test-Path -Path .\baseline.txt

    if ($baselineExists) {
        # Delete it
        Remove-Item -Path .\baseline.txt
    }
}


Write-Host ""
Write-Host "What would you like to do?"
Write-Host ""
Write-Host "    A) Collect new Baseline?"
Write-Host "    B) Begin monitoring files with saved Baseline?"
Write-Host ""
$response = Read-Host -Prompt "Please enter 'A' or 'B'"
Write-Host ""

if ($response -eq "A".ToUpper()) {
    # Delete baseline.txt if it already exists
    Erase-Baseline-If-Already-Exists

    # Calculate Hash from the target files and store in baseline.txt
    # Collect all files in the target folder
    $files = Get-ChildItem -Path .\Files

    # For each file, calculate the hash, and write to baseline.txt
    foreach ($f in $files) {
        $hash = Calculate-File-Hash $f.FullName
        "$($hash.Path)|$($hash.Hash)" | Out-File -FilePath .\baseline.txt -Append
    }
    
}

elseif ($response -eq "B".ToUpper()) {
    
    $fileHashDictionary = @{}

    # Load file|hash from baseline.txt and store them in a dictionary
    $filePathsAndHashes = Get-Content -Path .\baseline.txt
    
    foreach ($f in $filePathsAndHashes) {
         $fileHashDictionary.add($f.Split("|")[0],$f.Split("|")[1])
    }

    # Begin (continuously) monitoring files with saved Baseline
    while ($true) {
        Start-Sleep -Seconds 1
        
        $files = Get-ChildItem -Path .\Files

        # For each file, calculate the hash, and write to baseline.txt
        foreach ($f in $files) {
            $hash = Calculate-File-Hash $f.FullName
            #"$($hash.Path)|$($hash.Hash)" | Out-File -FilePath .\baseline.txt -Append

            # Notify if a new file has been created
            if ($fileHashDictionary[$hash.Path] -eq $null) {
                # A new file has been created!
                Write-Host "$($hash.Path) has been created!" -ForegroundColor Green
            }
            else {

                # Notify if a new file has been changed
                if ($fileHashDictionary[$hash.Path] -eq $hash.Hash) {
                    # The file has not changed
                }
                else {
                    # File file has been compromised!, notify the user
                    Write-Host "$($hash.Path) has changed!!!" -ForegroundColor Yellow
                }
            }
        }

        foreach ($key in $fileHashDictionary.Keys) {
            $baselineFileStillExists = Test-Path -Path $key
            if (-Not $baselineFileStillExists) {
                # One of the baseline files must have been deleted, notify the user
                Write-Host "$($key) has been deleted!" -ForegroundColor DarkRed -BackgroundColor Gray
            }
        }
    }
}

