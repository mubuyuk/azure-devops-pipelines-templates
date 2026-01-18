# Master Pipeline – Orkestrering av child pipelines

Detta repo innehåller en **master-pipeline i Azure DevOps** vars syfte är att **starta flera befintliga pipelines i andra repos från en central plats**.

Pipelinen innehåller **ingen build-, test- eller deploy-logik**.  
Den fungerar enbart som en **orkestrator / kontrollpunkt**.

---

## Lösning – vad denna pipeline gör

Denna master-pipeline:

- körs manuellt
- läser en konfigurationsfil (`pipelines.json`)
- triggar flera andra pipelines via **Azure DevOps REST API**
- startar dem **oberoende av varandra**
- loggar:
  - vilken pipeline som startats
  - RunId
  - klickbar URL till respektive pipeline-run

Resultatet är:
> **Ett klick → flera pipelines startas**

---

## Vad denna pipeline INTE gör

Pipelinen:

- väntar **inte** på att child-pipelines ska bli klara
- stoppar **inte** andra pipelines om en misslyckas
- innehåller **ingen** logik för test, build eller deploy
- skapar **inga beroenden** mellan pipelines

Varje child-pipeline ansvarar fullt ut för sitt eget flöde.

---

## pipelines.json – konfiguration

`pipelines.json` innehåller **vilka pipelines som ska startas**.

### Exempel

```json
[
  {
    "name": "repo-app-a",
    "pipelineId": 10
  },
  {
    "name": "repo-app-b",
    "pipelineId": 11
  },
  {
    "name": "repo-app-c",
    "pipelineId": 12
  }
]
```

### Fält

- **name**  
  Ett läsbart namn som används i loggar och sammanfattning

- **pipelineId**  
  Azure DevOps pipeline-id som används för att trigga rätt pipeline via API

### Lägga till ny pipeline

1. Skapa pipelinen i Azure DevOps  
2. Lägg till ett nytt objekt i `pipelines.json`  

Ingen kod i master-pipelinen behöver ändras.

---

## Hur master-pipelinen fungerar (översikt)

### 1. Manuell start

```yaml
trigger: none
```

Pipelinen körs endast när någon aktivt väljer **Run pipeline**.

---

### 2. Exekveringsmiljö

```yaml
pool:
  vmImage: 'ubuntu-latest'
```

Pipelinen körs på en **Microsoft-hostad Linux-agent**.

---

### 3. Azure DevOps-kontext

```powershell
$organization = "$(System.CollectionUri)"
$project      = "$(System.TeamProject)"
```

Används för att bygga korrekta REST-API-URL:er.

---

### 4. Läsa pipeline-konfiguration

```powershell
$pipelines = Get-Content "pipelines.json" | ConvertFrom-Json
```

Separering av **konfiguration** och **logik**.

---

## Trigga pipelines via REST API

Master-pipelinen använder **Azure DevOps REST API** istället för manuella klick.

### REST-anrop

```text
POST /_apis/pipelines/{pipelineId}/runs
```

Detta skapar en ny pipeline-run i Azure DevOps.

### Kodexempel

```powershell
$response = Invoke-RestMethod `
    -Method Post `
    -Uri $url `
    -Headers $headers `
    -Body "{}"
```

### Autentisering

```powershell
Authorization = "Bearer $(System.AccessToken)"
```

> OBS: **Allow scripts to access OAuth token** måste vara aktiverad.

### Officiell dokumentation

https://learn.microsoft.com/en-us/rest/api/azure/devops/pipelines/runs/run-pipeline

---

## Spårning och sammanfattning

För varje startad pipeline sparas:

- namn
- pipelineId
- runId
- web-url

Exempel från loggen:

```text
App:        repo-app-a
PipelineId: 10
RunId:      67
Url:        https://dev.azure.com/...
```

---

## Sammanfattning

Denna master-pipeline fungerar som:

> **En central kontrollpunkt för att starta flera självständiga pipelines**

Den förenklar manuellt arbete, skalar över tid och skapar bättre överblick
utan att skapa hårda beroenden mellan pipelines.
