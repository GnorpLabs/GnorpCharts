# Unofficial Astroneer helm chart

## Traefik Ingress

To use the traefik ingress it assumes you have the associated entrypoint already created via a traefik deployment.

## Auto-generated server secret

To get the server password secret that has been generated, use the following command in a terminal that is authenticated with the kubernetes cluster. For the console password it is the same command just change 'serverPassword' to 'rconPassword' and 'astroneer-password' to 'astroneer-rcon-password'

windows:
`kubectl get secret -n astroneer astroneer-password -o jsonpath='{.data.serverPassword}' | %{[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_))}`

linux:
`kubectl get secret -n astroneer astroneer-password -o jsonpath='{.data.serverPassword}' | base64 -d`

## Config

Used environment variables to create the AstroServerSettings.ini so secrets can be auto generated added to ini via init container.

configmap version:

```yaml
astroserversettings:
  path: /astrotux/AstroneerServer/Astro/Saved/Config/WindowsServer/AstroServerSettings.ini
  config: |
    [/Script/Astro.AstroServerSettings]
    bLoadAutoSave=true
    MaxServerFramerate=30
    MaxServerIdleFramerate=3.000000
    bWaitForPlayersBeforeShutdown=false
    ConsolePassword=
    PublicIP=
    ServerName=
    ServerGuid=
    OwnerName=
    OwnerGuid=
    PlayerActivityTimeout=0
    ServerPassword=
    bDisableServerTravel=false
    DenyUnlistedPlayers=false
    VerbosePlayerProperties=false
    AutoSaveGameInterval=900
    BackupSaveGamesInterval=7200
    ActiveSaveFileDescriptiveName=SAVE_1
    ShareCode=
    ServerAdvertisedName=
```
