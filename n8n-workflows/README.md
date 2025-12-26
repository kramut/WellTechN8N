# n8n Workflows - WellTech Hub

Questa cartella contiene i workflow n8n esportati per l'automazione WellTech Hub.

## Come importare i workflow

1. Accedi a n8n: `https://welltechn8n-production.up.railway.app`
2. Vai su "Workflows"
3. Clicca "Import from File" (o il pulsante "+" → "Import from File")
4. Seleziona il file JSON che vuoi importare
5. Il workflow verrà importato e potrai modificarlo

## Workflow disponibili

### 01-trend-analysis.json
**Trend Analysis Weekly** - Analizza trend settimanali da Google Trends e salva nel database.

**Nodi:**
- Manual Trigger
- Setup Keywords (Code)
- SerpAPI Google Trends
- Process Results (Code)
- Aggregate Trends
- Save Trends to Backend (HTTP Request)

**Configurazione richiesta:**
- Credenziali SerpAPI configurate in n8n
- URL Backend: `https://welltechbackend-production.up.railway.app`

**Come usare:**
1. Importa il workflow
2. Configura le credenziali SerpAPI nel nodo "SerpAPI Google Trends"
3. Verifica che l'URL del backend sia corretto
4. Esegui manualmente per testare
5. (Opzionale) Cambia Manual Trigger con Schedule Trigger per esecuzione automatica

## Note

- I workflow potrebbero richiedere piccole modifiche dopo l'importazione (es. credenziali, URL)
- Verifica sempre i nodi dopo l'importazione
- Testa ogni workflow prima di attivarlo in produzione

