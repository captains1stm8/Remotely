# Remotely server deployment notes (for Claude Code running on the VPS)

Target: build and run `Remotely_Server` on this VPS behind nginx with TLS.

## Facts about this codebase (verified)
- Snapshot of upstream Remotely from 2019-06. Server targets **netcoreapp2.2** (EOL 2019-12-23). Current distros do not package the 2.2 runtime, so **do not install .NET on the host**. Build and run in Docker using `deploy/Dockerfile` (`mcr.microsoft.com/dotnet/core/sdk:2.2-bionic` → `aspnet:2.2-bionic`).
- **Do not run `Utilities/Remotely_Server_Install.sh`.** It targets Ubuntu 18.04 only, installs `aspnetcore-runtime-2.2` from the 18.04 apt repo, and downloads a prebuilt zip from `remotely.lucency.co` instead of using this source.
- Only `Remotely_Server` + `Remotely_Shared` are needed for the server. The `*.Win` projects are .NET Framework 4.7.2/WPF and cannot build on Linux. Compiled JS for `wwwroot/scripts/*.ts` is committed, so no TypeScript/npm step is needed.
- Kestrel listens on `127.0.0.1:5000`. The container runs with `--network host`, so nginx proxies to loopback and the default ForwardedHeaders loopback trust applies (no `KnownProxies` change needed).
- DB: SQLite at `/data/Server.db` (Docker volume `remotely-data`). EF migrations run automatically at startup (`Startup.cs`, `context.Database.Migrate()`).
- SignalR hubs: `/BrowserHub`, `/DeviceHub`, `/RCBrowserHub`, `/RCDeviceHub`. They need WebSocket upgrade, which `deploy/remotely.nginx.conf` provides.

Verified in a test environment (2026-10-21): image builds; `/`, `/Identity/Account/Register` and `/scripts/Main.js` return 200; `Server.db` is created and migrated; `nginx -t` passes on the config. **Not** verified: TLS, real agent connections, session recording.

## Steps
Run from the repo root. Requires root or sudo.

1. Install Docker and nginx if they are missing: `apt-get install -y docker.io nginx` (Debian/Ubuntu).
2. Build: `docker build -f deploy/Dockerfile -t remotely-server .`
   - Expected warnings: NU1608 (Microsoft.CodeAnalysis version constraint). Harmless.
3. Run:
   ```bash
   docker run -d --name remotely --restart unless-stopped --network host \
     -v remotely-data:/data remotely-server
   curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:5000/   # expect 200
   ```
4. nginx: copy `deploy/remotely.nginx.conf` to `/etc/nginx/sites-available/remotely`, replace `remotely.example.com` with the real domain, symlink it into `sites-enabled`, then run `nginx -t && systemctl reload nginx`.
5. TLS: DNS for the domain must already point at the VPS, and ports 80/443 must be open (ufw, and the OVH Network Firewall if it is enabled in the OVH control panel). Then run `apt-get install -y certbot python3-certbot-nginx && certbot --nginx -d <domain>`.
6. After TLS, set `ApplicationOptions__RedirectToHTTPS=true` and `ApplicationOptions__UseHSTS=true` as container env vars (`-e`), then recreate the container.
7. Register the first account in the browser. Then disable self-registration by recreating the container with `-e ApplicationOptions__AllowSelfRegistration=false`.

## Configuration
- Override any `appsettings.json` key with an env var, using `__` for `:`. Example: `-e ApplicationOptions__SmtpHost=smtp.example.com`. `WebHost.CreateDefaultBuilder` loads env vars after the json files.
- Recreate the container to change env (data survives in the volume): `docker rm -f remotely && docker run ...` (same command as step 3, plus the `-e` flags).
- Logs: `docker logs -f remotely`.
- Back up the DB: `docker run --rm -v remotely-data:/data -v "$PWD":/b ubuntu:24.04 cp /data/Server.db /b/`.

## Known gaps
- `wwwroot/Downloads` contains only the install scripts. The agent and desktop zips/exes they reference are produced by `Utilities/Publish.ps1`, which needs Windows for the .NET Framework projects. Agent downloads from this server will 404 until those are built and copied into the container's `/app/wwwroot/Downloads`.
- Session recording (`RecordRemoteControlSessions`) is untested. The code starts the literal `ffmpeg.exe`, so the image symlinks `/usr/local/bin/ffmpeg.exe` → ffmpeg. Recordings are written under `/app` and are lost when the container is recreated unless you mount a volume.
- .NET Core 2.2 and the bionic base images get no security patches. Don't expose port 5000 directly. Keep it behind nginx on loopback.
