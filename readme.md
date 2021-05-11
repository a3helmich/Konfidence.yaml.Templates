Some yml templates I use on my azure devops server. Needed by some published projects. Keep in mind that currently you cannot reference github from azure devops <ins>server</ins>, the templates need to be in the same organisation on devops server.

- GenerateAndPublishCodeCoverage.yml: needs package 'dotnet tool update -g dotnet-reportgenerator-globaltool'