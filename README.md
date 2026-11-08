# <img src="assets/logo.svg" width="40" height="40"/> Simple Sync Project

Simple Sync Project aims to bring you a simple way of synchronizing data between Playnite instances. Everything is
self-hosted and kept on your own machine.

## Description

The project allows you to synchronize data between multiple Playnite instances running at the same time. It allows you
to use multiple Playnite instances on different devices while maintaining a single shared library.

**Currently only games and library objects such as tags, platforms, and genres are supported. Game install data,
emulators, ROMs are currently not synchronized. Other extensions are currently not supported either.**

## Modules

The project consists of three modules:

* [Plugin](https://github.com/Yalgrin/playnite-simple-sync-plugin) - Playnite plugin for two-way synchronization of data
  between the Playnite instance and the server.
* [Server](https://github.com/Yalgrin/playnite-simple-sync-server) - Self-hosted server for storing and coordinating
  data synchronization between plugin client and a management panel.
* [Panel](https://github.com/Yalgrin/simple-sync-panel) - Management panel for browsing and editing data stored on the
  server.

## Future plans

- installation data synchronization - optional, either shared by some clients or completely separate
- emulator and ROM support
- support for some Playnite extensions
- Playnite 11 Alpha support
- extension synchronization/migration
- panel UI improvements (more responsive, maybe a mobile version)
- better client and user management

## Instructions

If you plan to use multiple Playnite instances, it is recommended to start with only a single instance with a non-empty
library. For every additional client you can either start with a fresh instance or you can copy over entire Playnite
folder from %APPDATA% directory.

You can still use multiple instances with filled libraries, but **be warned that you will likely lose some data e.g.
extension data for games existing in multiple instances**. After doing a complete sync, you should be fine.

**Make backups before doing anything**.

### Recommended setup - single compose file with all components

1. Download the [
   `example-compose.yml`](https://raw.githubusercontent.com/Yalgrin/simple-sync-project/refs/heads/master/example-compose.yml)
   file.
2. The only required piece of configuration are locations of the server data and logs. You can change them in the
   following lines:

```yaml
  volumes:
    - 'D:/Playnite-Sync-Server/metadata/:/app/metadata'
    - 'D:/Playnite-Sync-Server/logs/:/app/logs'

  volumes:
    - 'D:/Playnite-Sync-Server/postgres/data/:/var/lib/postgresql/data'
```

Replace `D:/Playnite-Sync-Server` with the path to the directory of your choosing.

3. Navigate to the directory where `example-compose.yml` is located and execute the following command in the
   terminal/command line:

```
docker compose -f example-compose.yml up -d
```

The server should now be running on port 8093 (unless changed). You can now specify the URL of the app in the plugin
settings as follows:

```
http://<SERVER_IP>:8093
```

Replace `<SERVER_IP>` with the IP address of the machine that's running the application. You can manually check if it's
running by opening the following link:

```
http://<SERVER_IP>:8093/api/health
```

The panel should be running on port 8094 (unless changed). Open it in your browser using the following link and try
logging in:

```
http://<SERVER_IP>:8094
```

The panel forwards all requests to the server, so you can specify the panel URL in the plugin.

**The default username and password for the panel are `admin` and `admin`. Change it after logging in with an option in
top right of the webpage.**

### Migration from old compose format (without panel)

1. Download the [
   `example-compose.yml`](https://raw.githubusercontent.com/Yalgrin/simple-sync-project/refs/heads/master/example-compose.yml)
   file.
2. Add to your existing compose.yml the `playnite-sync-panel` section:

```yaml
  playnite-sync-panel:
    image: yalgrin/playnite-simple-sync-panel:<version>
    container_name: playnite-simple-sync-panel
    #...the rest
```

3. Reapply changes to the compose file.

```
docker compose -f your-compose.yml up -d
```

### Plugin setup

1. Install the plugin either manually or through Playnite addon manager.
2. Install all the necessary library extensions.
3. Enable synchronization and configure the sync server address in the plugin settings (Add-ons → Extensions settings →
   Generic → Simple Sync). Test the connection to make sure Playnite can reach the server. Make sure other options are
   unchecked.
4. If you've previously synced your library, fetch the data using the option in the main menu (Main menu → Extensions →
   Simple Sync → Fetch everything).
5. Send your library data to the sync server using the option in the main menu (Main menu → Extensions → Simple Sync →
   Synchronize everything).
6. Go back to the plugin settings and configure other options as you wish. For continuous synchronization, enable all
   three check-boxes.

## Compatibility table

The following table shows which versions of each module is compatible with each other. From version 0.2.0 onwards the
plugin has a version check mechanism to make sure the currently used version of the server is compatible.

| Playnite version | Plugin version | Server version | Panel version |
|------------------|----------------|----------------|---------------|
| 10.x             | 0.4.0          | 0.4.0          | 0.1.0         |
| 10.x             | 0.3.x          | 0.3.x          | N/A           |
| 10.x             | 0.2.x          | 0.2.x          | N/A           |
| 10.x             | 0.1.x          | 0.1.x          | N/A           |

## Reverse proxy setup

If you want to put the apps behind a reverse proxy (for HTTPS for instance), you will need to disable buffering for a
few specific endpoints. Here's an example configuration for NGINX:

```
    location /api/client/connect {
        proxy_pass <url>;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Connection "";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 1h;
    }

    location /api/panel/change/connect {
        proxy_pass <url>;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Connection "";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 1h;
    }

    location / {
        proxy_pass <url>;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
```

## Known issues

#### Default completion statuses are gone after fetching data from the sync server

Due to the way the syncing process works and because you cannot change default completion statuses with Playnite SDK,
you might find that default completion statuses are gone after fetching data from the sync server. You should receive a
notification when this happens. To resolve this, simply go to the Library Manager and assign those values.

#### Playnite fails to connect to the sync server

Make sure that the sync server is running and that the address in the plugin settings is correct. It's possible that the
port is being blocked by your firewall.

## Troubleshooting

If you encounter any issues, please check the following:

- server logs (probably by using `docker logs`)
- Playnite extensions log (extensions.log in %appdata%/Playnite folder)
- Network tab from DevTools in your browser (when facing panel problems, look for failed requests or errors in the
  console)

When reporting bugs, please provide all of the above information, as well as:

- Playnite version,
- exact versions of the plugin, server and/or panel,
- steps to reproduce the issue.

## Donate

If you like what I do, feel free to buy me a coffee.

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/yalgrin)

## Credits

- [Playnite](https://playnite.link/) by [JosefNemec](https://github.com/JosefNemec)
