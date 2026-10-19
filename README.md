# Brief di sviluppo — Season Wanted (versione reale, a fasi)

## Contesto

Season Wanted è una piattaforma in cui i fan di anime dimostrano quanto vogliono una nuova stagione, un sequel, un remake o un film. Prima misura l'interesse (senza denaro), poi, solo se il titolare dei diritti accetta, apre un vero crowdfunding "tutto o niente".

Allego `index.html`, una demo single-page. Usala **solo come riferimento di design, testi e flussi**: non riusare il suo codice. Nella demo i dati sono inventati, l'autenticazione è solo nel browser e le password sono salvate male.

## Regole di lavoro

1. Lavora **una fase alla volta**. A fine fase fermati, riassumi cosa hai fatto, spiega come provarlo e aspetta la mia conferma.
2. TypeScript ovunque, codice commentato, nessun segreto nel codice (solo variabili d'ambiente, con un `.env.example`).
3. Nessun dato finto in produzione. Dove serve una demo, mostra un banner "versione di prova".
4. Se una scelta riguarda legge, pagamenti o minori, non decidere tu: segnalala nella sezione "Decisioni aperte".

## Stack

- **Frontend e server:** Next.js (App Router) + TypeScript + Tailwind
- **Database, login, storage:** Supabase (PostgreSQL + Auth + Row Level Security)
- **Email:** Resend (o SMTP di Supabase)
- **Hosting:** Vercel
- **Fase 1: niente** Redux, Redis, Docker o test E2E. Si aggiungono solo se servono.

Motivo: Supabase offre già login, verifica email, reset password e database sicuro, così non scriviamo da zero la parte dove si sbaglia di più.

---

## Fase 1 — MVP senza denaro

**Obiettivo:** capire se le persone usano davvero la piattaforma.

### Account
- Registrazione con email e password tramite Supabase Auth (nessun hash fatto a mano).
- Verifica email e reset password veri.
- Dati: nome utente, data di nascita, accettazione di Termini e Privacy (salva versione e data).
- Sotto i 14 anni: registrazione bloccata **dal server**. Sotto i 18: account consentito, ma niente contributi.
- Sessione con cookie `httpOnly` (non salvare token in `localStorage`).
- Eliminazione account: anonimizza i dati personali, conserva solo ciò che la legge richiede.

### Database (tabelle principali)
`profiles`, `anime`, `campaigns`, `interests`, `comments`, `likes`, `favorites`, `reports`, `admin_actions` (audit log).

Regole obbligatorie:
- `UNIQUE (user_id, campaign_id)` su `interests`: un solo contributo dichiarato per account e campagna.
- `CHECK` sull'importo (1–50 €).
- Contatori (sostenitori, totale, medio, like, preferiti) **calcolati dal database**, mai inviati dal client.
- Row Level Security attiva su ogni tabella.
- Stati campagna gestiti dal backend: `draft → pending_review → published → threshold_reached → rights_verification → official → funding_active → successful | failed | cancelled`.

### Funzioni utente
- Home con campagne, filtri (più sostenuti, in crescita, quasi alla soglia, recenti).
- Ricerca unica per anime e account.
- Pagina campagna: descrizione, contatori, mi piace, commenti, preferiti, bottone "Sono interessato" con importo dichiarato (€1/2/3/5/altro) e avviso che **non è un pagamento**.
- Classifica sostenitori per campagna con podio, **solo per chi ha scelto di essere visibile** (opt-in, di default nascosto).
- Proposta di una nuova richiesta: testo personale obbligatorio, va in coda di moderazione, mai pubblicata in automatico. Spiegare che la piattaforma fa da ponte con gli studi e che la proposta viene controllata.
- Segnalazione di campagne e commenti (spam, truffa, copyright, falso).
- Profilo con follower, seguiti, contributi.

### Admin (area separata, solo ruolo admin)
- Approva, rifiuta o chiedi modifiche alle proposte.
- Gestione segnalazioni, blocco utenti, sospensione campagne.
- **Audit log** di ogni azione (chi, cosa, quando).
- Statistiche base: utenti, campagne, interessi, segnalazioni.

### Sicurezza
- Validazione di tutti gli input sul server (es. Zod).
- Rate limiting su login, registrazione, interesse, commenti e segnalazioni.
- Nessun `dangerouslySetInnerHTML`; output sempre escapato.
- Protezione CSRF: cookie `SameSite`, controllo dell'origine, Server Actions.
- Header di sicurezza e nessun segreto nel client.
- Filtro base per insulti sui commenti, lato server.

### SEO
Pagina pubblica per ogni anime e campagna, con titolo, descrizione e anteprima social dinamici, più `sitemap.xml` e `robots.txt`.

### Criteri di accettazione (Fase 1)
- Un utente si registra, verifica l'email, accede ed esce.
- Un account non può dichiarare interesse due volte sulla stessa campagna, nemmeno modificando le richieste a mano.
- Un minorenne non può dichiarare contributi, anche modificando la pagina.
- Le proposte non compaiono finché un admin non le approva.
- I contatori restano corretti dopo molte operazioni e non dipendono dal client.
- Ogni azione admin compare nell'audit log.

---

## Fase 2 — Parte social (solo se la Fase 1 funziona)

- Community per categoria e per anime, con chat.
- Chat con filtro anti-insulti, segnala e blocca utente.
- Follow tra utenti.
- Notifiche email per milestone delle campagne seguite.
- Sezione "Guarda": solo video **incorporati dai canali ufficiali**, senza caricamenti propri (servono licenze).

## Fase 3 — Studi verificati

- Account studio con verifica **manuale** di identità, società, contatti e titolarità dei diritti.
- Spunta blu assegnata **solo da un admin**, dopo la verifica.
- Dashboard per gli studi: domanda aggregata, crescita, Paesi, commenti ricorrenti.
- Report "Fan Demand" da inviare ai titolari dei diritti.

## Fase 4 — Pagamenti (solo dopo consulenza legale e fiscale)

Prima di scrivere codice servono le decisioni della sezione "Decisioni aperte".

Poi, in modalità test:
- Provider specializzato (es. Stripe Connect) con onboarding e verifica di chi riceve i fondi.
- Nessun dato di carta nel nostro sistema (Checkout o Elements del provider).
- Tabella `transactions` con stati e storico.
- Webhook firmati e idempotenti: il server conferma il pagamento, mai il client.
- Importi calcolati dal server.
- Formula "tutto o niente" e rimborsi. Attenzione: le autorizzazioni sulle carte scadono dopo circa 7 giorni, quindi non bastano per campagne lunghe. Va scelto un modello compatibile.
- Commissione e costi di pagamento mostrati prima della conferma, senza schermate ingannevoli.

## Fase 5 — Qualità e messa online

- Test unitari sulla logica (conteggi, stati, permessi) e test end-to-end sui flussi critici (registrazione, interesse, moderazione).
- CI su GitHub Actions: controlli e test a ogni modifica.
- Backup del database, monitoraggio errori, controllo accessibilità.
- Privacy Policy, Termini, Cookie Policy e regole della community scritte da un professionista.

---

## Decisioni aperte (da confermare con me prima di iniziare)

1. I €0,99 per proporre una richiesta: vanno rimborsati se la proposta viene rifiutata?
2. Visibilità dei contributi nelle classifiche: opt-in, come indicato sopra?
3. Immagini e locandine degli anime: come ottenere i permessi? Per ora solo testo e colori.
4. Età minima per i pagamenti e gestione dei minori.
5. Paesi e lingue del lancio.
6. Struttura giuridica del crowdfunding (donazione, ricompensa, altro) e commissione.

## Cose da NON fare

- Token di accesso salvati in `localStorage`.
- Calcoli di contributi o totali sul client.
- Salvare dati di carta.
- Eliminare account con soft delete senza anonimizzazione.
- Caricare video di studi senza licenza.
- Promettere che l'anime verrà prodotto al raggiungimento della soglia.

## Cosa consegnare a ogni fase

1. Struttura delle cartelle e spiegazione breve.
2. Migrazioni SQL e regole di sicurezza del database.
3. `.env.example` con tutte le variabili.
4. Istruzioni per avviare il progetto in locale.
5. Elenco di test manuali da fare per verificare la fase.

