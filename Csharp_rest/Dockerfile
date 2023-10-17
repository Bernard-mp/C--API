# Build stage
FROM mcr.microsoft.com/dotnet/sdk:6.0-alpine AS build
WORKDIR /App

# Copy the application source code into the build image
COPY . ./

# Restore dependencies and publish the application
RUN dotnet restore
RUN dotnet publish -c Release -o out

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:6.0-alpine
WORKDIR /MyApp

# Copy the published application from the build image
COPY --from=build /App/out .

# Expose the port the application will listen on (port 5000)
EXPOSE 5000

# Define the entry point for the application
ENTRYPOINT ["dotnet", "Csharp_rest.dll"]
