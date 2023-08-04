---
layout: post
title: Retrieving values of sensitive variables in Octopus Deploy
tags: powershell octopus
---
When working with variables in Octopus Deploy, you want to keep passwords or API keys secret and out of sight.  Octopus offers "sensitive variables" for this use case. Sensitive variables are stored encrypted in the database and cannot be retrieved once set. In the UI or deployment logs the variables will only appear as \*\*\*\*\*\*\*.
If, like me, you manage to set a sensitive variable and forget to save it to your personal secret vault, how can you retrieve the contents of the variable? Let's say you want the value of the sensitive variable called "password". The easiest way to retrieve the value of the sensitive variable is to create a runbook in the project where the variable is stored.  Create a script step and set it to run on a desired worker, as we don't want anything to run on a deployment target.  We want to write the 'password' in clear text to a text file by adding the following script to the script step.

```powershell
"password: $password" | Out-File C:\temp\password.txt
```

Run the runbook, connect to the machine it was run on and open password.txt. The password will be logged in clear text and you can save the password. 

```txt
password: Pa$$w0rd
```
