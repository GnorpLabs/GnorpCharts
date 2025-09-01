# Unofficial Astroneer helm chart

## Config

Used environment variables to create the ini so secrets can be auto generated added to ini via init container.

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
