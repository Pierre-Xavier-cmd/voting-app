FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY Worker.csproj .
RUN dotnet restore
COPY Program.cs .
RUN dotnet publish -c release -o /app/publish --no-restore

FROM mcr.microsoft.com/dotnet/runtime:7.0 AS runtime
WORKDIR /app
RUN useradd -m -u 1000 appuser
COPY --from=build /app/publish .
RUN chown -R appuser:appuser /app
USER appuser
ENTRYPOINT ["dotnet", "Worker.dll"]