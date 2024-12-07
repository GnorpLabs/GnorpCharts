![version: 0.1.2](https://img.shields.io/badge/version-0.1.2-informational?style=flat-square) ![type: application](https://img.shields.io/badge/type-application-informational?style=flat-square) ![app version: v1.6.1](https://img.shields.io/badge/app%20version-v1.6.1-informational?style=flat-square) [![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/satisfactory)](https://artifacthub.io/packages/search?repo=satisfactory)

# satisfactory
Helm chart for deploying a Satisfactory server in Kubernetes. This chart can be installed without modification to deploy a Satisfactory server using the defaults specified in the [wolveix/satisfactory-server](https://hub.docker.com/r/wolveix/satisfactory-server) image.

# Installation

```sh
helm repo add 98jan https://98jan.github.io/helm-charts/
helm install satisfactory 98jan/satisfactory
```

## Setup

Recent updates consume 4GB - 6GB RAM, but [the official wiki](https://satisfactory.wiki.gg/wiki/Dedicated_servers#Requirements) recommends allocating 12GB - 16GB RAM.

## Environment Variables

| Parameter               |  Default  | Function                                            |
|-------------------------|:---------:|-----------------------------------------------------|
| `AUTOPAUSE`             |  `true`   | pause game when no player is connected              |
| `AUTOSAVEINTERVAL`      |   `300`   | autosave interval in seconds                        |
| `AUTOSAVENUM`           |    `5`    | number of rotating autosave files                   |
| `AUTOSAVEONDISCONNECT`  |  `true`   | autosave when last player disconnects               |
| `CRASHREPORT`           |  `true`   | automatic crash reporting                           |
| `DEBUG`                 |  `false`  | for debugging the server                            |
| `DISABLESEASONALEVENTS` |  `false`  | disable the FICSMAS event (you miserable bastard)   |
| `MAXOBJECTS`            | `2162688` | set the object limit for your server                |
| `MAXPLAYERS`            |    `4`    | set the player limit for your server                |
| `MAXTICKRATE`           |   `30`    | set the maximum sim tick rate for your server       |
| `NETWORKQUALITY`        |    `3`    | set the network quality/bandwidth for your server   |
| `PGID`                  |  `1000`   | set the group ID of the user the server will run as |
| `PUID`                  |  `1000`   | set the user ID of the user the server will run as  |
| `ROOTLESS`              |  `false`  | run the container as a non-root user                |
| `SERVERBEACONPORT`      |  `15000`  | set the game's beacon port                          |
| `SERVERGAMEPORT`        |  `7777`   | set the game's port                                 |
| `SERVERIP`              | `0.0.0.0` | set the game's ip (usually not needed)              |
| `SERVERQUERYPORT`       |  `15777`  | set the game's query port                           |
| `SKIPUPDATE`            |  `false`  | avoid updating the game on container start/restart  |
| `STEAMBETA`             |  `false`  | set experimental game version                       |
| `TIMEOUT`               |   `30`    | set client timeout (in seconds)                     |

## Experimental Branch

If you want to run a server for the Experimental version of the game, set the `STEAMBETA` environment variable to `true`.

## How to Improve the Multiplayer Experience

The [Satisfactory Wiki](https://satisfactory.wiki.gg/wiki/Multiplayer#Engine.ini) recommends a few config tweaks to really get the best out of multiplayer. These changes are already applied to the server, but they need to be applied to your local config too:

-   Press `WIN + R`
-   Enter `%localappdata%/FactoryGame/Saved/Config/WindowsNoEditor`
-   Copy the config data from the wiki into the respective files
-   Right-click each of the 3 config files (Engine.ini, Game.ini, Scalability.ini)
-   Go to Properties > tick Read-only under the attributes
