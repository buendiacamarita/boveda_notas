##### - Ruta del perfil de PowerShell:
C:\Users\<TuUsuario>\Documents\PowerShell\Microsoft.PowerShell_profile.ps1

##### - Cómo crear/editar el perfil:
$PROFILE
###### Si no existe:
if (!(Test-Path $PROFILE)) { New-Item -Path $PROFILE -ItemType File -Force }

###### Ejemplo de Alias:
Set-Alias -Name ll -Value Get-ChildItem
Set-Alias -Name .. -Value "Set-Location .."
Set-Alias -Name c -Value Clear-Host
Set-Alias -Name edit-profile -Value "code $PROFILE"

##### - Donde guardar scripts para poder usarlos en la terminal
1. Crea una carpeta para tus scripts, por ejemplo: C:\Scripts\PowerShell
2. Abre PowerShell como **administrador**.
3. [System.Environment]::SetEnvironmentVariable("PATH", $env:PATH + ";C:\Scripts\PowerShell", "User")
4. **Reinicia PowerShell** para aplicar cambios.

##### - Ejemplo de Script:
```
param (
    [string]$ProjectName
)
 $projectPath = "C:\dev\$ProjectName"
if (Test-Path $projectPath) {
    Set-Location $projectPath
    code .
} else {
    Write-Host "El proyecto '$ProjectName' no existe en C:\dev"
}
```
