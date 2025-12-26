# 🚀 Piano di Implementazione - Workflow Automatici WellTech Hub

**Versione:** 2.0 (Approccio Ibrido)  
**Data:** 2024  
**Status:** Pronto per implementazione

---

## 📑 Indice

1. [Architettura Generale](#architettura-generale)
2. [Fase 1: Setup Infrastruttura](#fase-1-setup-infrastruttura)
3. [Fase 2: Trend Analysis Settimanale](#fase-2-trend-analysis-settimanale)
4. [Fase 3: Ricerca e Pre-selezione Prodotti (Automatica)](#fase-3-ricerca-e-pre-selezione-prodotti-automatica)
5. [Fase 4: Dashboard Approvazione Prodotti (Ibrida)](#fase-4-dashboard-approvazione-prodotti-ibrida)
6. [Fase 5: Inserimento Prodotti Approvati (Automatica)](#fase-5-inserimento-prodotti-approvati-automatica)
7. [Fase 6: Scrittura Articoli SEO](#fase-6-scrittura-articoli-seo)
8. [Fase 7: Aggiornamento Pathfinder](#fase-7-aggiornamento-pathfinder)
9. [Fase 8: Generazione Video AI](#fase-8-generazione-video-ai)
10. [Fase 9: Post Automatici Social](#fase-9-post-automatici-social)
11. [Fase 10: Workflow Master Orchestrato](#fase-10-workflow-master-orchestrato)
12. [Stack Tecnologico](#stack-tecnologico)
13. [Costi Stimati](#costi-stimati)
14. [Timeline Implementazione](#timeline-implementazione)

---

## 🏗️ Architettura Generale

```
welltech-workflows/
├── docs/
│   └── IMPLEMENTATION_PLAN.md    # Questo documento
├── n8n-workflows/                # Workflow n8n (JSON exports)
│   ├── 01-trend-analysis.json
│   ├── 02-product-search.json
│   ├── 03-article-generation.json
│   ├── 04-video-generation.json
│   ├── 05-social-posting.json
│   └── 00-master-workflow.json
├── scripts/                      # Script Node.js per logica complessa
│   ├── affiliate-apis/
│   │   ├── awin-client.ts
│   │   ├── clickbank-client.ts
│   │   └── amazon-client.ts
│   ├── ai-content/
│   │   ├── article-generator.ts
│   │   └── video-script-generator.ts
│   └── utils/
│       ├── seo-optimizer.ts
│       └── pathfinder-updater.ts
├── integrations/                 # Wrapper API per servizi esterni
│   ├── openai.ts
│   ├── did.ts
│   └── cloudinary.ts
├── config/                       # Configurazioni
│   ├── affiliate-credentials.json.example
│   └── ai-config.json.example
└── README.md
```

---

## 📦 Fase 1: Setup Infrastruttura

### 1.1 Scelta Piattaforma Workflow

**Raccomandazione:** n8n self-hosted su Railway

**Vantaggi:**
- Controllo completo
- Costi contenuti (~$10-20/mese)
- Open-source
- Estendibile

**Alternativa:** Make (cloud, più user-friendly ma costoso)

### 1.2 Setup n8n su Railway

```bash
# Crea docker-compose.yml in welltech-workflows/
version: '3.8'
services:
  n8n:
    image: n8nio/n8n
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=${N8N_PASSWORD}
      - N8N_HOST=${N8N_HOST}
      - N8N_PROTOCOL=https
    volumes:
      - n8n_data:/home/node/.n8n
volumes:
  n8n_data:
```

### 1.3 API Endpoints Backend da Aggiungere

Aggiungere al backend `welltech-api/src/routes/workflows.ts`:

```typescript
// Trend analysis
GET  /api/workflows/trends              // Lista trend recenti
POST /api/workflows/trends/analyze      // Analizza trend settimanali

// Product candidates (pre-approvazione)
GET  /api/workflows/products/candidates // Lista prodotti candidati
POST /api/workflows/products/candidates // Crea candidato
PUT  /api/workflows/products/candidates/:id/approve  // Approva → inserisce in DB
PUT  /api/workflows/products/candidates/:id/reject   // Rifiuta

// Article generation
POST /api/workflows/articles/generate   // Genera articolo da prodotto
PUT  /api/workflows/articles/:id/publish // Pubblica articolo

// Pathfinder
POST /api/workflows/pathfinder/update   // Aggiorna mapping percorsi
```

### 1.4 Schema Database da Aggiungere

Aggiungere a `prisma/schema.prisma`:

```prisma
model Trend {
  id          Int       @id @default(autoincrement())
  keyword     String
  source      String    // "google", "reddit", "amazon"
  score       Float
  category    String?
  metadata    Json?     // Dati aggiuntivi
  createdAt   DateTime  @default(now())
  
  @@map("trends")
}

model ProductCandidate {
  id                  Int       @id @default(autoincrement())
  name                String
  category            String
  description         String?   @db.Text
  price               Decimal?  @db.Decimal(10, 2)
  affiliateLink       String    @db.Text
  affiliateProgram    String    // "awin", "clickbank", "amazon"
  commissionPercentage Decimal? @db.Decimal(5, 2)
  imageUrl            String?   @db.Text
  rating              Float?
  reviewCount         Int?
  source              String    // "awin", "clickbank", etc.
  sourceId            String?   // ID nel sistema sorgente
  status              String    @default("pending") // "pending", "approved", "rejected"
  rejectionReason     String?   @db.Text
  approvedAt          DateTime?
  approvedBy          String?   // User ID o email
  metadata            Json?     // Dati aggiuntivi
  createdAt           DateTime  @default(now())
  updatedAt           DateTime  @updatedAt
  
  @@map("product_candidates")
}

model PathfinderMapping {
  id          Int       @id @default(autoincrement())
  goalId      String    // "feel-better", "sexual-wellbeing", etc.
  stepOrder   Int
  articleId   Int?
  productId   Int?
  relevance   Float     // Score 0-1
  createdAt   DateTime  @default(now())
  
  article     Article?  @relation(fields: [articleId], references: [id])
  product     Product?  @relation(fields: [productId], references: [id])
  
  @@map("pathfinder_mappings")
}
```

---

## 📊 Fase 2: Trend Analysis Settimanale

### 2.1 Fonti Trend

- **Google Trends API** (via Pytrends o API ufficiale)
- **Reddit API** (r/wellness, r/productivity, r/fitness)
- **Amazon Best Sellers** (scraping o API)
- **Twitter/X API** (hashtag monitoring)

### 2.2 Workflow n8n

```
Trigger: Cron (ogni lunedì 9:00)
  ↓
1. Google Trends Check
   - Query: ["wellness", "productivity", "fitness", "sexual wellness", "sustainability"]
   - Regione: IT, US, UK
   - Timeframe: ultimi 7 giorni
   - Output: Top 20 keyword per regione
  ↓
2. Reddit Scraper
   - Subreddits: r/wellness, r/productivity, r/fitness, r/sex, r/environment
   - Estrai: Top posts ultima settimana
   - Keywords: Titoli + commenti top
  ↓
3. Amazon Best Sellers
   - Categorie: Health, Fitness, Self-Help, Sexual Wellness
   - Estrai: Top 20 prodotti per categoria
   - Keywords: Nome prodotto + categoria
  ↓
4. Aggrega Risultati
   - Combina trend da tutte le fonti
   - Ranking per rilevanza (score combinato)
   - Rimuovi duplicati
  ↓
5. Salva in Database
   - POST /api/workflows/trends/analyze
   - Salva top 50 trend nella tabella `trends`
  ↓
6. Notifica Telegram/Email
   - Report settimanale con top 10 trend
   - Formato: Keyword | Score | Source | Categoria
```

### 2.3 Output Esempio

```json
{
  "trends": [
    {
      "keyword": "sexual wellness",
      "score": 8.5,
      "sources": ["google", "reddit"],
      "category": "sexual-wellbeing",
      "growth": "+25%"
    },
    {
      "keyword": "fotovoltaico domestico",
      "score": 7.2,
      "sources": ["google", "amazon"],
      "category": "sustainability",
      "growth": "+18%"
    }
  ]
}
```

---

## 🔍 Fase 3: Ricerca e Pre-selezione Prodotti (Automatica)

### 3.1 Approccio Ibrido

**Automatico:**
- Ricerca prodotti via API affiliate
- Pre-selezione con filtri automatici
- Generazione link affiliato
- Estrazione dati prodotto

**Manuale:**
- Review e approvazione prodotti
- Controllo qualità e allineamento brand

### 3.2 Integrazioni API Affiliate

#### Awin API
- **Endpoint:** `https://api.awin.com`
- **Funzionalità:**
  - Lista merchant/programmi
  - Ricerca prodotti
  - Generazione link affiliato
  - Commissioni e performance
- **Documentazione:** https://wiki.awin.com/index.php/Advertiser_API

#### ClickBank API
- **Endpoint:** `https://api.clickbank.com`
- **Funzionalità:**
  - Ricerca prodotti
  - Generazione link
  - Commissioni
- **Documentazione:** https://support.clickbank.com/hc/en-us/articles/229313627

#### Amazon Associates API
- **Endpoint:** `https://webservices.amazon.com`
- **Funzionalità:**
  - Product Advertising API
  - Ricerca prodotti
  - Generazione link
  - Commissioni
- **Documentazione:** https://webservices.amazon.com/paapi5/documentation/

### 3.3 Script Node.js: `scripts/affiliate-apis/awin-client.ts`

```typescript
// Esempio struttura
class AwinClient {
  async searchProducts(keyword: string, category?: string): Promise<ProductCandidate[]>
  async getProductDetails(productId: string): Promise<ProductDetails>
  async generateAffiliateLink(productId: string): Promise<string>
  async getCommissionRate(merchantId: string): Promise<number>
}
```

### 3.4 Workflow n8n - Ricerca Prodotti

```
Trigger: Settimanale (dopo trend analysis) O Manuale
  ↓
1. Input: Trend/Categoria
   - Prende top 5 trend dalla settimana
   - O input manuale: categoria specifica
  ↓
2. Per ogni trend:
   
   a. Awin Product Search
      - Query: keyword del trend
      - Filtri: commissione > 5%, rating > 4.0
      - Limite: 20 prodotti
   
   b. ClickBank Product Search
      - Query: keyword del trend
      - Filtri: gravity > 50, commissione > 30%
      - Limite: 20 prodotti
   
   c. Amazon Product Search
      - Query: keyword del trend
      - Filtri: rating > 4.0, reviews > 100
      - Limite: 20 prodotti
  ↓
3. Validazione Automatica
   - Check duplicati (confronta con DB prodotti esistenti)
   - Filtra per commissione minima (configurabile)
   - Valuta rating e recensioni
   - Estrai immagine prodotto (scraping o API)
  ↓
4. Genera Link Affiliato
   - Per ogni prodotto: genera link affiliato via API
   - Salva link + tracking ID
  ↓
5. Crea Product Candidates
   - POST /api/workflows/products/candidates (bulk)
   - Status: "pending"
   - Include: tutti i dati + link affiliato
  ↓
6. Notifica per Review
   - Email/Telegram con:
     * Numero prodotti trovati
     * Link dashboard approvazione
     * Preview top 5 prodotti
```

### 3.5 Filtri Automatici Configurabili

```typescript
const FILTERS = {
  minCommission: 5.0,        // Commissione minima %
  minRating: 4.0,            // Rating minimo (1-5)
  minReviews: 50,            // Recensioni minime
  maxPrice: 500,             // Prezzo massimo (opzionale)
  allowedCategories: [...],  // Categorie permesse
  blockedMerchants: [...],   // Merchant bloccati
};
```

---

## ✅ Fase 4: Dashboard Approvazione Prodotti (Ibrida)

### 4.1 Interfaccia Web

Creare pagina frontend: `welltech-frontend/app/admin/products/candidates/page.tsx`

**Funzionalità:**
- Lista prodotti candidati (status: "pending")
- Preview prodotto: nome, immagine, prezzo, commissione, rating
- Filtri: categoria, commissione, rating
- Azioni: **Approva** | **Rifiuta** | **Modifica**

### 4.2 Componente React: `ProductCandidateCard.tsx`

```typescript
// Struttura componente
<ProductCandidateCard
  candidate={candidate}
  onApprove={(id) => approveProduct(id)}
  onReject={(id, reason) => rejectProduct(id, reason)}
  onEdit={(id) => editProduct(id)}
/>
```

### 4.3 Endpoint Backend

```typescript
// GET /api/workflows/products/candidates
// Query params: ?status=pending&category=...

// PUT /api/workflows/products/candidates/:id/approve
// Body: { notes?: string }
// Action: Crea prodotto in tabella `products`, elimina candidato

// PUT /api/workflows/products/candidates/:id/reject
// Body: { reason: string }
// Action: Aggiorna status = "rejected", salva reason
```

### 4.4 Workflow Approvazione

```
Trigger: Click "Approva" nella dashboard
  ↓
1. Validazione Finale
   - Check link affiliato valido
   - Verifica immagine disponibile
  ↓
2. Crea Prodotto
   - POST /api/products (con dati candidato)
   - Genera slug da nome
  ↓
3. Elimina Candidato
   - DELETE /api/workflows/products/candidates/:id
  ↓
4. Trigger Workflow Articolo
   - Avvia generazione articolo per prodotto (Fase 6)
```

---

## 📥 Fase 5: Inserimento Prodotti Approvati (Automatica)

### 5.1 Endpoint Backend: `PUT /api/workflows/products/candidates/:id/approve`

```typescript
// Logica nel controller
async function approveCandidate(req, res) {
  const candidate = await getCandidate(req.params.id);
  
  // Crea prodotto
  const product = await createProduct({
    name: candidate.name,
    category: candidate.category,
    description: candidate.description,
    price: candidate.price,
    affiliateLink: candidate.affiliateLink,
    affiliateProgram: candidate.affiliateProgram,
    commissionPercentage: candidate.commissionPercentage,
    imageUrl: candidate.imageUrl,
  });
  
  // Elimina candidato
  await deleteCandidate(candidate.id);
  
  // Trigger workflow articolo (opzionale)
  await triggerArticleGeneration(product.id);
  
  return res.json({ product, message: 'Prodotto approvato e inserito' });
}
```

### 5.2 Notifica Inserimento

```
Email/Telegram:
"✅ Prodotto approvato: [Nome]
   Categoria: [Categoria]
   Commissione: [X]%
   Link: [Preview]
   
   Prossimo step: Generazione articolo SEO in corso..."
```

---

## ✍️ Fase 6: Scrittura Articoli SEO

### 6.1 AI Writing Tool

**Raccomandazione:** OpenAI GPT-4

**Alternativa:** Claude (Anthropic)

### 6.2 Workflow n8n

```
Trigger: Prodotto approvato e inserito
  ↓
1. Input: Prodotto ID
   - Recupera dati prodotto dal backend
  ↓
2. Ricerca Keyword SEO
   - Google Keyword Planner (via API)
   - Ubersuggest/SEMrush (se disponibile)
   - Genera: keyword principale + 5-10 long-tail
  ↓
3. Genera Outline Articolo
   - GPT-4 Prompt: "Crea outline SEO per [prodotto]"
   - Include: H1, H2, H3, meta description
   - Lunghezza target: 2000 parole
  ↓
4. Scrittura Contenuto
   - GPT-4: Scrive articolo completo
   - Sezioni: intro, features, benefits, FAQ, CTA
   - Include link prodotto affiliate
  ↓
5. Ottimizzazione SEO
   - Check keyword density (2-3%)
   - Aggiungi internal links (articoli correlati)
   - Genera slug ottimizzato
   - Meta description: 155 caratteri
  ↓
6. Genera Immagine Featured
   - DALL-E 3: Genera immagine basata su prodotto
   - O usa immagine prodotto esistente
   - Upload su Cloudinary
  ↓
7. Crea Articolo nel Backend
   - POST /api/workflows/articles/generate
   - Status: "draft" (per review opzionale)
   - O "published" (pubblicazione automatica)
  ↓
8. Notifica
   - Email: "Articolo generato: [Titolo]"
   - Link preview articolo
```

### 6.3 Prompt Template GPT-4

```
Scrivi un articolo SEO ottimizzato di 2000 parole su [PRODOTTO].

Contesto:
- Nome prodotto: [NOME]
- Categoria: [CATEGORIA]
- Prezzo: [PREZZO]
- Link affiliato: [LINK]

Requisiti SEO:
- Keyword principale: [KEYWORD]
- Long-tail keywords: [KEYWORD1], [KEYWORD2], [KEYWORD3]
- Target audience: [AUDIENCE]
- Tone: informativo ma coinvolgente

Struttura:
1. Introduzione (200 parole) - Hook + problema
2. Cos'è [PRODOTTO] (300 parole) - Descrizione
3. Caratteristiche principali (400 parole) - Features
4. Benefici (400 parole) - Perché sceglierlo
5. Come funziona (300 parole) - Utilizzo
6. FAQ (300 parole) - 5 domande comuni
7. Conclusione + CTA (100 parole) - Link affiliato

Formato: Markdown con H1, H2, H3
Meta description: 155 caratteri, include keyword principale
```

### 6.4 Script Node.js: `scripts/ai-content/article-generator.ts`

```typescript
async function generateArticle(product: Product, keywords: string[]): Promise<Article> {
  const prompt = buildPrompt(product, keywords);
  const content = await openai.chat.completions.create({
    model: "gpt-4",
    messages: [{ role: "user", content: prompt }],
    temperature: 0.7,
  });
  
  const article = parseContent(content.choices[0].message.content);
  return optimizeSEO(article, keywords);
}
```

---

## 🗺️ Fase 7: Aggiornamento Pathfinder

### 7.1 Logica Aggiornamento

Quando viene creato un nuovo articolo/prodotto:
- Analizza categoria e keywords
- Matcha con obiettivi pathfinder esistenti
- Calcola rilevanza per ogni percorso
- Aggiorna mapping nel database

### 7.2 Workflow n8n

```
Trigger: Articolo pubblicato
  ↓
1. Analizza Contenuto
   - Estrai: categoria, keywords, tags
   - NLP: sentiment analysis, topic modeling
  ↓
2. Match con Percorsi
   - Confronta con goalOptions esistenti
   - Calcola rilevanza per ogni percorso (0-1)
   - Filtra: rilevanza > 0.6
  ↓
3. Determina Step Ottimale
   - Per ogni percorso matchato:
     * Analizza step esistenti
     * Trova step più rilevante (per categoria/keyword)
     * O crea nuovo step se necessario
  ↓
4. Aggiorna Pathfinder Mapping
   - POST /api/workflows/pathfinder/update
   - Salva in tabella `pathfinder_mappings`
   - Include: goalId, stepOrder, articleId, relevance
  ↓
5. Notifica (opzionale)
   - "Articolo aggiunto a X percorsi"
```

### 7.3 Script Node.js: `scripts/utils/pathfinder-updater.ts`

```typescript
async function updatePathfinderMappings(article: Article): Promise<void> {
  const goals = await getRelevantGoals(article.category, article.keywords);
  
  for (const goal of goals) {
    const stepOrder = await findOptimalStep(goal.id, article);
    const relevance = calculateRelevance(article, goal);
    
    await createPathfinderMapping({
      goalId: goal.id,
      stepOrder,
      articleId: article.id,
      relevance,
    });
  }
}
```

---

## 🎬 Fase 8: Generazione Video AI

### 8.1 Servizi Video AI

**Raccomandazione:** D-ID (buon rapporto qualità/prezzo)

**Alternativa:** Synthesia (qualità alta, costoso)

### 8.2 Workflow n8n

```
Trigger: Articolo pubblicato
  ↓
1. Genera Script Video
   - GPT-4: Crea script 60-90 secondi
   - Formato: hook, problema, soluzione, CTA
   - Tone: engaging, TikTok-style
  ↓
2. Genera Video con Avatar AI
   - D-ID API: Crea video con avatar
   - Voice: Italiano, voce naturale
   - Background: Branded (WellTech colors)
   - Dimensioni: 1080x1920 (verticale)
  ↓
3. Aggiungi Overlay
   - Logo WellTech (angolo)
   - Testo chiave (keyword principale)
   - Link prodotto (QR code opzionale)
  ↓
4. Export Video
   - Formato: MP4, 1080x1920
   - Durata: 60-90 secondi
   - Upload su Cloudinary
  ↓
5. Salva Video nel Backend
   - POST /api/videos
   - Link articolo associato
   - Salva videoUrl
  ↓
6. Preparazione Social
   - Genera thumbnail
   - Estrai caption (da script)
   - Hashtag suggeriti
```

### 8.3 Script Template Video

```
[Hook - 5s]
"Stai cercando [PROBLEMA]? Ho trovato la soluzione perfetta!"

[Problema - 15s]
Spiega il problema che risolve il prodotto in modo coinvolgente

[Soluzione - 30s]
Presenta il prodotto, features principali, benefici

[Social Proof - 10s]
"Già migliaia di persone lo usano con risultati incredibili"

[CTA - 10s]
"Link in bio per scoprire di più e approfittare dello sconto!"
```

---

## 📱 Fase 9: Post Automatici Social

### 9.1 Piattaforme

- **TikTok API** (richiede approvazione)
- **Instagram Graph API** (Business Account)
- **YouTube Shorts API** (via YouTube Data API)

### 9.2 Workflow n8n

```
Trigger: Video generato
  ↓
1. Preparazione Contenuto
   - Video: già generato
   - Caption: da script articolo (max 2200 char)
   - Hashtag: generati automaticamente (10-15)
   - Thumbnail: generato o estratto
  ↓
2. Post su TikTok
   - TikTok API: Upload video
   - Aggiungi caption + hashtag
   - Pubblica immediatamente o programma
  ↓
3. Post su Instagram Reels
   - Instagram Graph API: Upload Reel
   - Stessa caption/hashtag
   - Tag prodotto (se disponibile)
  ↓
4. Post su YouTube Shorts
   - YouTube Data API: Upload Short
   - Titolo: keyword principale
   - Descrizione: link articolo + affiliate
   - Tags: hashtag generati
  ↓
5. Aggiorna Backend
   - PUT /api/videos/:id
   - Salva: tiktokUrl, instagramUrl, youtubeUrl
  ↓
6. Monitoraggio (24h dopo)
   - Check views su TikTok/Instagram
   - Aggiorna tiktokViews nel DB
   - Notifica se performance > soglia
```

### 9.3 Hashtag Generator

```typescript
function generateHashtags(category: string, keyword: string): string[] {
  const base = ['#welltech', '#benessere', '#wellness', '#italia'];
  
  const categoryTags = {
    'sexual-wellbeing': ['#benesseresessuale', '#vitalità', '#intimità'],
    'sustainability': ['#sostenibilità', '#green', '#fotovoltaico'],
    'wellbeing': ['#salute', '#vitalità', '#energia'],
    // ...
  };
  
  const keywordTag = `#${keyword.replace(/\s+/g, '')}`;
  
  return [...base, ...categoryTags[category] || [], keywordTag].slice(0, 15);
}
```

---

## 🎯 Fase 10: Workflow Master Orchestrato

### 10.1 Workflow Principale Settimanale

```
Trigger: Cron (ogni lunedì 9:00)
  ↓
┌─────────────────────────────────────────┐
│ FASE 1: Trend Analysis                  │
│ - Esegui Workflow Trend Analysis        │
│ - Output: Top 10 trend settimanali     │
└─────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────┐
│ FASE 2: Ricerca Prodotti                │
│ - Per ogni trend: cerca prodotti        │
│ - Output: 50-100 prodotti candidati    │
└─────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────┐
│ FASE 3: Notifica per Approvazione      │
│ - Email: "X prodotti pronti per review" │
│ - Link dashboard approvazione           │
└─────────────────────────────────────────┘
  ↓
[PAUSA - Attesa approvazione manuale]
  ↓
┌─────────────────────────────────────────┐
│ FASE 4: Per ogni prodotto approvato:   │
│                                         │
│  a. Inserimento automatico (Fase 5)     │
│  b. Genera Articolo SEO (Fase 6)        │
│  c. Aggiorna Pathfinder (Fase 7)       │
│  d. Genera Video AI (Fase 8)            │
│  e. Post Social (Fase 9)               │
└─────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────┐
│ FASE 5: Report Settimanale              │
│ - Articoli creati: X                    │
│ - Video generati: Y                     │
│ - Post pubblicati: Z                    │
│ - Trend coperti: ...                    │
│ - Email riepilogo                       │
└─────────────────────────────────────────┘
```

### 10.2 Workflow Manuale (On-Demand)

```
Trigger: Manuale (dashboard admin)
  ↓
Input: Categoria o keyword specifica
  ↓
Esegui: Ricerca prodotti → Approvazione → Articolo → Video → Social
```

---

## 🛠️ Stack Tecnologico

### Workflow Automation
- **n8n** (self-hosted su Railway) - $10-20/mese
- Alternativa: Make (cloud) - $9-29/mese

### AI & Content
- **OpenAI GPT-4** - $50-200/mese (a seconda volume)
- **D-ID** (video avatar) - $29-99/mese
- **DALL-E 3** (immagini) - Incluso in OpenAI

### Storage & Media
- **Cloudinary** (video, immagini) - $0-49/mese (free tier generoso)
- **S3** (backup) - $5-10/mese

### APIs Affiliate
- **Awin API** - Gratis
- **ClickBank API** - Gratis
- **Amazon Associates API** - Gratis

### APIs Esterne
- **Google Trends API** - Gratis (via Pytrends)
- **Reddit API** - Gratis
- **TikTok API** - Gratis (richiede approvazione)
- **Instagram Graph API** - Gratis (Business Account)
- **YouTube Data API** - Gratis

### Database
- **PostgreSQL** (già presente)
- **Redis** (cache, queue) - $5-10/mese (opzionale)

### Totale Stimato: ~$100-400/mese

---

## 💰 Costi Stimati

### MVP (Fase 1-5)
- n8n: $10-20/mese
- OpenAI GPT-4: $30-50/mese
- Cloudinary: $0 (free tier)
- **Totale: ~$40-70/mese**

### Completo (Fase 1-10)
- n8n: $10-20/mese
- OpenAI GPT-4: $50-200/mese
- D-ID: $29-99/mese
- Cloudinary: $0-49/mese
- Redis: $5-10/mese
- **Totale: ~$100-400/mese**

---

## 📅 Timeline Implementazione

### Settimana 1-2: Setup Infrastruttura
- [ ] Setup n8n su Railway
- [ ] Creare endpoint backend per workflow
- [ ] Aggiungere schema database (Trend, ProductCandidate, PathfinderMapping)
- [ ] Setup credenziali API affiliate (Awin, ClickBank, Amazon)

### Settimana 3-4: Trend Analysis e Ricerca Prodotti
- [ ] Implementare Workflow 2 (Trend Analysis)
- [ ] Creare script Awin/ClickBank/Amazon client
- [ ] Implementare Workflow 3 (Ricerca prodotti)
- [ ] Test ricerca e pre-selezione automatica

### Settimana 5-6: Dashboard Approvazione
- [ ] Creare pagina frontend dashboard approvazione
- [ ] Implementare endpoint approvazione/rifiuto
- [ ] Implementare Workflow 4 (Dashboard)
- [ ] Test flusso approvazione → inserimento

### Settimana 7-8: Generazione Contenuti
- [ ] Implementare Workflow 6 (Articoli SEO)
- [ ] Setup OpenAI GPT-4 integration
- [ ] Test generazione articoli
- [ ] Implementare Workflow 7 (Pathfinder update)

### Settimana 9-10: Video e Social
- [ ] Setup D-ID integration
- [ ] Implementare Workflow 8 (Video AI)
- [ ] Setup TikTok/Instagram API
- [ ] Implementare Workflow 9 (Post social)
- [ ] Test end-to-end

### Settimana 11-12: Orchestrazione e Ottimizzazione
- [ ] Implementare Workflow 10 (Master workflow)
- [ ] Test workflow completo settimanale
- [ ] Ottimizzazione performance
- [ ] Documentazione finale

---

## 📝 Note Implementazione

### Sicurezza
- Credenziali API in variabili d'ambiente
- Autenticazione per endpoint workflow (API key)
- Rate limiting per API esterne

### Monitoring
- Logging workflow (n8n built-in)
- Error tracking (Sentry opzionale)
- Notifiche errori (Telegram/Email)

### Scalabilità
- Queue system per task pesanti (Bull/BullMQ)
- Batch processing per inserimenti multipli
- Caching per API calls frequenti (Redis)

---

## 🚀 Prossimi Passi Immediati

1. **Setup n8n su Railway**
   - Creare account Railway
   - Deploy n8n con docker-compose
   - Configurare variabili d'ambiente

2. **Creare Endpoint Backend**
   - Aggiungere `routes/workflows.ts`
   - Implementare endpoint base
   - Test con Postman/curl

3. **Setup Credenziali API**
   - Registrarsi a Awin, ClickBank, Amazon Associates
   - Ottenere API keys
   - Salvare in `.env`

4. **Test Workflow Base**
   - Workflow 3 (Ricerca prodotti) manuale
   - Verificare output e formattazione

---

**Documento creato:** 2024  
**Ultimo aggiornamento:** 2024  
**Versione:** 2.0 (Approccio Ibrido)

---

## 📚 Risorse Utili

- [n8n Documentation](https://docs.n8n.io/)
- [Awin API Docs](https://wiki.awin.com/index.php/Advertiser_API)
- [ClickBank API Docs](https://support.clickbank.com/hc/en-us/articles/229313627)
- [OpenAI API Docs](https://platform.openai.com/docs)
- [D-ID API Docs](https://docs.d-id.com/)




