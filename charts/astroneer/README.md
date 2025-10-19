# Unofficial Astroneer helm chart

## Traefik Ingress

To use the traefik ingress it assumes you have the associated entrypoint already created via a traefik deployment.

## Auto-generated server secret

To get the server password secret that has been generated, use the following command in a terminal that is authenticated with the kubernetes cluster. For the console password it is the same command just change 'serverPassword' to 'rconPassword' and 'astroneer-password' to 'astroneer-rcon-password'

windows:
`kubectl get secret -n astroneer astroneer-password -o jsonpath='{.data.serverPassword}' | %{[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_))}`

linux:
`kubectl get secret -n astroneer astroneer-password -o jsonpath='{.data.serverPassword}' | base64 -d`

## Configs

Used environment variables to create the AstroServerSettings.ini so secrets can be auto generated added to ini via init container.

launcher.toml and engine.ini have not been implemented as I believed having a server password was more critical.

```yaml
launchersettings: # TODO: Implement launcher settings
  path: /astrotux/launcher.toml
  config: # must have a prefix of 'launcher_' to be picked up by init container
    launcher_AutoUpdateServer: true
    launcher_CheckNetwork: true
    launcher_OverwritePublicIP: false
    launcher_notifications:
      method: ""
      name: "Astroneer Dedicated Server"
      EventWhitelist:
        - message
        - start
        - registered
        - shutdown
        - crash
        - player_join
        - player_leave
        - command
        - save
        - savegame_change
    launcher_status:
      SendStatus: false
      Interval: 120
      EndpointURL: ""
    launcher_LogDebugMessages: true
    launcher_AstroServerPath: "/astrotux/AstroneerServer"
    launcher_OverrideWinePath: "/opt/proton/files/bin/wine64"
    launcher_WinePrefixPath: "/astrotux/winepfx"
    launcher_WineBootTimeout: 30
    launcher_LogPath: "/astrotux/logs"
    launcher_PlayfabAPIInterval: 2
    launcher_ServerStatusInterval: 3.0
    launcher_DisableEncryption: false
    launcher_WrapperPath: ""
```

```yaml
astroserversettings:
  path: /astrotux/AstroneerServer/Astro/Saved/Config/WindowsServer/AstroServerSettings.ini
  config: # must have a prefix of 'astroneer_' to be picked up by init container
    astroneer_bLoadAutoSave: true
    astroneer_MaxServerFramerate: 30
    astroneer_MaxServerIdleFramerate: 3.000000
    astroneer_bWaitForPlayersBeforeShutdown: false
    astroneer_PublicIP: "xxx.xxx.xxx.xxx"
    astroneer_ServerName: "Change Me Goo Goo"
    astroneer_ServerGuid: "" # TODO: Autogenerate hexadecimal serverguid, this MUST be SET or xAuth token will error
    astroneer_OwnerName: "" # first person to join steam name
    astroneer_OwnerGuid: "" # set to 0 before first person joins
    astroneer_PlayerActivityTimeout: 0
    astroneer_bDisableServerTravel: false
    astroneer_DenyUnlistedPlayers: false
    astroneer_VerbosePlayerProperties: false
    astroneer_AutoSaveGameInterval: 900
    astroneer_BackupSaveGamesInterval: 7200
    astroneer_ActiveSaveFileDescriptiveName: SAVE_1
    astroneer_ShareCode: ""
    astroneer_ServerAdvertisedName: ""
    # astroneer_MaximumPlayerCount: 8 # default
    # astroneer_ConsolePort: 1234 # default
    # astroneer_VerbosePlayerProperties: false # this is automatically enforced by astroTux
    # astroneer_HeartbeatInterval: 55 # this is automatically enforced by astroTux
    # astroneer_ServerPassword will be added from generated secret or values.secret ref
    # astroneer_ConsolePassword will be added from generated secret or values.rcon.secret ref
```

```yaml
enginesettings: # TODO: implement engine.ini config options
  path: /astrotux/AstroneerServer/Astro/Saved/Config/WindowsServer/Engine.ini
  config: # must have a prefix of 'engine_' to be picked up by init container
    engine_bLoadAutoSave: true
    engine_Port: 7777
    engine_AllowEncryption: true
    engine_Paths: "[]"
    engine_MaxClientRate: 1000000
    engine_MaxInternetClientRate: 1000000
```
