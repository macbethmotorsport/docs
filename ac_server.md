# AC Server Install Notes

### Reference Material

default steam install guide:

* https://steamcommunity.com/app/244210/discussions/0/2828702373004724010/
* https://github.com/p3t3c/AssettoCorsaLinuxScripts/blob/master/acServer.sh
* https://github.com/germanrcuriel/assetto-corsa-server
* https://hub.docker.com/r/seejy/assetto-server-manager
  * https://github.com/cj123/assetto-server-manager
* https://github.com/jo3stevens/ACServerManager/blob/master/README_Linux.md

### create a new steam account w/stimguard disabled

 AC Server is free, so... this prevents private crediantals from being jacked.

 https://store.steampowered.com/join

 new steam account to use:
 u: `DicksonDillweed`
 p: `Butterflakes40`

## VPS Setup

#### just run shit as root... yes i know what I'm doing

```bash
$ dpkg --add-architecture i386
$ apt update
$ apt install zlib1g:i386 lib32gcc1
$ mkdir -p /ac/inst ; cd /ac/inst
$ wget http://media.steampowered.com/client/steamcmd_linux.tar.gz
$ tar -xvf steamcmd_linux.tar.gz
$ ./steamcmd.sh \
  +@sSteamCmdForcePlatformType windows \
  +login ${STEAM_USER:=DicksonDillweed} \
  +force_install_dir /ac/Steam \
  +app_update 302550 \
  quit

$ mkdir /ac/inst/server-manager
$ cd /ac/inst/server-manager
$ wget https://github.com/cj123/assetto-server-manager/releases/download/v1.3.2/server-manager_v1.3.2.zip
$ unzip server-manager_v1.3.2.zip
$ mv linux /ac/bin/server-manager
```

Update config.yml:
```bash
$ cp /ac/bin/server-manager/config.yml /ac/bin/server-manager/config.yml.dist
$ cat <<-EOF
steam:
  username: DicksonDillweed
  password: Butterflakes40

  install_path: /ac/Steam
  executable_path: /ac/Steam/acServer
  force_update: false

http:
  hostname: 0.0.0.0:8772
  session_key: RANDOMLY_GENERATE_THIS
  server_manager_base_URL:

monitoring:
  enabled: true

store:
  type: boltdb

  path: server_manager.db

accounts:
  admin_password_override:

live_map:
  refresh_interval_ms: 500

server:
  run_on_start:

championships:
  recaptcha:
    site_key:
    secret_key:
EOF

```

## Upload Tracks and Cars manually

The tracks and cars provided by steam are... super minimal. There's like a few files, and apparently that should work, but it didn't for me. Uploading the actual track and car data was the only way to make things work properly.

**Mount /ac directory on localhost via sshfs**
`sudo sshfs larry:/ac /Volumes/butter -p 16046 -d -o reconnect,sshfs_debug,allow_root -f`

... or not since sshfs won't work on my locked down `sshd_config`. Moving on:

Copying raw files from my sim's assetto content directly via scp:

```bash
scp -r ks_* larry:/ac/Steam/content/tracks
scp -r ks_porsche_911_gt* larry:/ac/Steam/content/cars
```

### Create basic start/stop script for systemd

```bash
#!/usr/bin/env bash

start() {
  cd /ac/bin/server-manager
  ./server-manager
}


stop() {
    pkill -9 server-manager
}

status() {
    systemctl status acsm
}

case "$1" in
    'start')
            start
            ;;
    'stop')
            stop
            ;;
    'status')
            status
            ;;
    *)
            echo
            echo "Usage: $0 { start | stop | status }"
            echo
            exit 1
            ;;
esac

exit 0
```

### Create custom systemd start/stop script for `server-manager`

```bash
[Unit]
Description=Assetto Corsa Server Manager (acsm)
ConditionPathExists=/ac/bin/server-manager/config.yml

[Service]
Type=simple
PIDFile=/ac/bin/server-manager/acsm.pid
ExecStart=/ac/bin/acsm.sh
ExecReload=kill -HUP $MAINPID
ExecStop=kill -9 $MAINPID
KillMode=process
Restart=on-failure

[Install]
WantedBy=default.target
```

^ paste into `/lib/systemd/system/acsm.service`



Now reload systemd configs, start the daemon, check the status, and if all works as expected, enable the service to start on system boot.

```bash
# reload configs
systemctl daemon-reload

# start daemon
systemctl start acsm

# check status
systemctl status acsm

# if we want to enable for system boot
systemctl enable acsm
```

Successful status should look like this (yay!)

```bash
└─[17:04]# systemctl status acsm
● acsm.service - Assetto Corsa Server Manager (acsm)
   Loaded: loaded (/lib/systemd/system/acsm.service; disabled; vendor preset: enabled)
   Active: active (running) since Sun 2019-06-09 17:04:34 CEST; 6s ago
 Main PID: 19132 (bash)
   CGroup: /system.slice/acsm.service
           ├─19132 bash /ac/bin/acsm.sh start
           └─19133 ./server-manager

Jun 09 17:04:34 larry systemd[1]: Started Assetto Corsa Server Manager (acsm).
Jun 09 17:04:34 larry acsm.sh[19132]: time="2019-06-09T17:04:34+02:00" level=info msg="initialising Raven monitoring"
Jun 09 17:04:34 larry acsm.sh[19132]: time="2019-06-09T17:04:34+02:00" level=info msg="initialising Prometheus Monitoring"
Jun 09 17:04:35 larry acsm.sh[19132]: time="2019-06-09T17:04:35+02:00" level=info msg="starting assetto server manager on: 0.0.0.0:
```

### Log in to the Server Manager Web UI

http://larry.laps.rip:8772/login
**username**: `admin`
**password**: `servermanager (first login) -> workit (new)`

**servername**: full tilt racing
**pass**: monza

### Status: server works, races work, but modded cars or tracks cause crash

No good info in logs.

```apt
# add the acserver node to /etc/prometheus/prometheus.yml:
- job_name: ac_server
  static_configs:
    - targets: ['localhost:8772']
```

tunnel tcp:9090 to localhost, then hit it with http://localhost:9090 for the UI

fuck. acserver isn't exporting anything relevant to acserver at all. it's all basic system shit. ffs.

Giving up for the night. Posted the issue here for help:
https://www.racedepartment.com/threads/ac-server-manager.165662/page-14

TODO

1. [done] copy all tracks and cars
2. [done] run test race
3. [done] setup systemd script for server-manager
4. set up nginx reverse https proxy for access



----------



#### OLD / DELETE THIS SHIT

> Steam App ID `244210` is the full AC install.. which fails
> `Error! App '244210' state is 0x212 after update job.`

Steam> login <username>;
Steam> passwd
Steam> <check for emailed "Steam Guard code">
Steam> force_install_dir /ac/Steam
Steam> app_update 302550  
Steam> exit

alternatively

./steamcmd.sh \
  +@sSteamCmdForcePlatformType windows \
  +login ${STEAM_USER:=anonymous} ${STEAM_PASSWORD} \
  +force_install_dir $INSTALL_PATH \
  +app_update 302550 \
  +quit

Steam> login <username>;  
Steam> force_install_dir ./assetto  
Steam> app_update 302550  
Steam> exit

#### init script

https://github.com/p3t3c/AssettoCorsaLinuxScripts/blob/master/acServer.sh
