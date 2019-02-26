FROM microsoft/dotnet:2.2-aspnetcore-runtime
WORKDIR /app
EXPOSE 43300
ENV ASPNETCORE_URLS http://0.0.0.0:43300
COPY ./publish /app
ENTRYPOINT ["dotnet", "netcorek8s.dll"]
