# Builds and runs Remotely_Server on the .NET Core 2.2 images (EOL; host distros no longer package 2.2).
# Build from repo root: docker build -f deploy/Dockerfile -t remotely-server .
FROM mcr.microsoft.com/dotnet/core/sdk:2.2-bionic AS build
WORKDIR /src
COPY Remotely_Shared/ Remotely_Shared/
COPY Remotely_Server/ Remotely_Server/
RUN dotnet publish Remotely_Server/Remotely_Server.csproj -c Release -r linux-x64 --self-contained false -o /app

FROM mcr.microsoft.com/dotnet/core/aspnet:2.2-bionic
# libgdiplus/libc6-dev/ffmpeg: session recording. RemoteControlSessionRecorder starts "ffmpeg.exe", hence the symlink.
RUN apt-get update && apt-get install -y --no-install-recommends libgdiplus libc6-dev ffmpeg \
    && rm -rf /var/lib/apt/lists/* \
    && ln -s /usr/bin/ffmpeg /usr/local/bin/ffmpeg.exe
WORKDIR /app
COPY --from=build /app .
ENV ASPNETCORE_URLS=http://127.0.0.1:5000 \
    ASPNETCORE_ENVIRONMENT=Production \
    ConnectionStrings__SQLite="DataSource=/data/Server.db"
VOLUME /data
ENTRYPOINT ["dotnet", "Remotely_Server.dll"]
