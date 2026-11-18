# ==============================================================================
# ÉTAPE 1 : Restauration des dépendances (Optimisation du cache des couches)
# ==============================================================================
FROM mcr.microsoft.com/dotnet/sdk:8.0-noble AS restore
WORKDIR /src

# Copie uniquement des fichiers de projet pour mettre en cache le layer NuGet
COPY ["src/MonApi.Domain/MonApi.Domain.csproj", "src/MonApi.Domain/"]
COPY ["src/MonApi.Application/MonApi.Application.csproj", "src/MonApi.Application/"]
COPY ["src/MonApi.Infrastructure/MonApi.Infrastructure.csproj", "src/MonApi.Infrastructure/"]
COPY ["src/MonApi.API/MonApi.API.csproj", "src/MonApi.API/"]

RUN dotnet restore "src/MonApi.API/MonApi.API.csproj"

# ==============================================================================
# ÉTAPE 2 : Compilation et Publication
# ==============================================================================
FROM restore AS publish
COPY . .
WORKDIR "/src/src/MonApi.API"

RUN dotnet publish "MonApi.API.csproj" \
    -c Release \
    -o /app/publish \
    --no-restore \
    /p:UseAppHost=false

# ==============================================================================
# ÉTAPE 3 : Image Finale Durcie (Runtime minimaliste Chiseled + Non-Root)
# ==============================================================================
FROM mcr.microsoft.com/dotnet/aspnet:8.0-noble-chiseled AS final
WORKDIR /app

EXPOSE 8080

ENV ASPNETCORE_URLS=http://+:8080 \
    ASPNETCORE_ENVIRONMENT=Production \
    DOTNET_RUNNING_IN_CONTAINER=true \
    DOTNET_EnableDiagnostics=0

COPY --from=publish --chown=$APP_UID:$APP_UID /app/publish .

USER $APP_UID

ENTRYPOINT ["dotnet", "MonApi.API.dll"]