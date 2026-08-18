# Calcetto · Classifica ELO

App di classifica ELO per il calcetto balilla tra Paolo e i suoi amici/colleghi. Questo README esiste per il Claude (o lo sviluppatore) del futuro: leggi tutto prima di toccare il codice.

## Regole d'oro (leggere PRIMA di fare qualsiasi cosa)

1. **Il DB è in PRODUZIONE con dati veri.** Non eseguire mai script SQL distruttivi senza conferma esplicita di Paolo.
2. **`import-partite.sql` è OBSOLETO e PERICOLOSO: non eseguirlo MAI.** Fa TRUNCATE e reimporta partite di una vecchia migrazione. Eseguirlo oggi cancellerebbe i dati reali.
3. **Prima gli script SQL, poi il deploy dell'HTML.** Se l'HTML interroga una tabella/vista che non esiste ancora, l'app va in errore per tutti. È già successo una volta.
4. **La base di lavoro è sempre l'index.html che Paolo incolla in chat**, non versioni ricordate a memoria. Le versioni possono divergere: verificare cosa contiene davvero il file prima di patchare.
5. Nei testi rivolti a Paolo: **niente em dash**, risposte dirette e concise, in italiano.

## Stack e deploy

- **Un singolo `index.html`**: vanilla JS (ES module), niente build, niente framework. CSS inline nel file.
- **Backend: Supabase** (Postgres + Auth + Realtime).
  - URL: `https://gtxeacrydiullkyqanpu.supabase.co`
  - Chiave publishable nel client: `sb_publishable_G7n7aUW5IhECINFT3T2ybw_JrquwZq7` (è pensata per stare nel client, la sicurezza è nelle RLS policy).
- **Hosting: GitHub Pages.** File nella root del repo: `index.html`, `manifest.json`, `sw.js`, `icon-192.png`, `icon-512.png`, `apple-touch-icon.png`. È una PWA installabile.
- Modalità TV: `?tv=1` mostra solo la classifica a caratteri grandi (per un monitor), con realtime attivo.

## Modello dei permessi

- **Admin = utente autenticato** (email+password Supabase, signup DISABILITATO dal pannello). `isAdmin()` è solo `!!state.session`; le vere protezioni sono le RLS policy.
- **Anonimi possono:** leggere tutto, proporre partite (status `pending`), proporre giocatori (`player_requests`), occupare un posto libero nei tornei con iscrizioni aperte (update su `tournaments` in status `signup`).
- **Solo admin:** confermare/rifiutare partite e richieste, gestire rosa, ruoli, stagioni, tornei, ricalcolo, backup, undo.

## Schema DB

Tabelle principali (vedi gli script SQL per i dettagli):

- **`players`**: `id`, `name` (unique), `elo`, `wins`, `losses` (contatori di stagione), `role` (`difensore`/`attaccante`/`ibrido`, default ibrido).
- **`seasons`**: `id`, `name`, `active` (bool), `created_at`, `ended_at`, `ends_at` (scadenza prevista, facoltativa), `awards` (jsonb, premi calcolati alla chiusura).
- **`matches`**: `mode` (`1v1`/`2v2`), `team_a`/`team_b` (array di player id), `score_a`/`score_b` (null se senza punteggio), `winner_side` (`A`/`B`, solo senza punteggio), `season_id`, `status` (`pending`/`confirmed`), `created_at`, `confirmed_at`, `deltas` (jsonb id->delta), `pre` (jsonb id->elo prima della partita), `winners` (array), `boost` (numeric, default 1), `boosted` (bool LEGACY: se `boost` è null e `boosted` true, vale 1.25).
- **`season_results`**: snapshot dell'albo d'oro alla chiusura stagione (`season_id`, `player_id`, `name`, `elo`, `wins`, `losses`, `rank`).
- **`tournaments`**: `name`, `season_id`, `teams` (jsonb array di `{p1, p2}`, null = posto libero), `bracket` (jsonb), `format` (`elim`/`girone`), `rated` (bool, false = "solo per la gloria"), `status` (`signup`/`open`/`done`).
- **`player_requests`**: proposte di nuovi giocatori (`name`, `created_at`).
- **`player_totals`**: VISTA (non tabella) con vittorie/sconfitte di sempre per giocatore, calcolata da `matches.winners`. I contatori su `players` sono solo della stagione corrente.

### Script SQL, in ordine di esecuzione storica

1. `supabase-setup.sql` (base: players, matches, seasons)
2. `supabase-auth-update.sql` (RLS admin/anonimi, partite pending)
3. `supabase-seasons-update.sql` (stagioni pianificate, ends_at)
4. `supabase-noscore-update.sql` (partite senza punteggio)
5. `supabase-riallineamento.sql` (fix storici una tantum)
6. `supabase-ruoli.sql` (colonna role)
7. `supabase-boost.sql` poi `supabase-boost2.sql` (boost: il 2 sostituisce il booleano con la colonna numeric; il primo è superato)
8. `supabase-torneo-rated.sql` (colonna rated e status signup)
9. `supabase-season-end.sql` (awards jsonb)
10. `supabase-richieste-giocatori.sql` (tabella player_requests)
11. `supabase-iscrizioni-premi.sql` (policy per iscrizioni anonime ai tornei)
12. `supabase-totali.sql` (vista player_totals)
13. `import-partite.sql`: **NON ESEGUIRE, vedi regole d'oro**

Su un DB già in produzione servono solo gli script nuovi. `loadAll` è resiliente: se mancano `player_requests` o `player_totals` l'app funziona e mostra un toast che indica lo script da eseguire; se mancano le tabelle core mostra l'errore vero a schermo.

## Regole ELO (le costanti sono in cima al modulo JS)

- `K = 32` (Paolo ha chiesto esplicitamente di non cambiarlo), partenza 1000, attesa classica su base 400.
- **MOV** (margine di vittoria): `ln(diffGol + 1) * 2.2 / (diffElo * 0.001 + 2.2)`. Un 10-9 vale ~0.7x, un 10-0 ~2.4x. Il denominatore smorza i cappotti del favorito.
- **Senza punteggio**: fattore `NO_SCORE_BASE = 0.5` al posto del MOV (con lo stesso smorzamento), banner rosso persistente nel form.
- **Coppia (2v2)**: `rating = media - HANDICAP * |diff|` con `HANDICAP = 0.25`. Ripartizione del delta: vincendo, il piu debole della coppia prende di piu; perdendo, il piu forte perde di piu.
- **Ripetizioni**: dalla 3ª partita dello stesso giorno tra le stesse due squadre, delta x0.5 (`repetitionFactor`, calcolato sul `created_at`).
- **Boost x1.25 (`LINEUP_BOOST`)**: solo per i tavoli proposti da "Chi c'è oggi" e accettati col bottone "Usa" con formazione ESATTA (consumo per partita da `state.suggestedList`). Con 8+ presenti: due tavoli con persone distinte, il Tavolo 1 vale SEMPRE x1.25, il Tavolo 2 solo se porta incroci/coppie nuove. Sotto gli 8: bonus solo se porta novità. NOTA: esisteva anche un bonus x1.4 per le "consigliate del giorno", RIMOSSO su richiesta di Paolo; le partite storiche che lo hanno ricevuto restano valide perché il boost è salvato sulla riga.
- **Suggeritore (`scorePairing`)**: novità (incroci mai visti x30, coppie mai viste x20), sinergia di ruoli SOLO come rompi-pareggi (ibrido+ibrido +3, ibrido+specialista +2.5, attaccante+difensore +2, doppio specialista dello stesso ruolo 0: nessuna coppia vietata o penalizzata, l'ibrido ha un filo di vantaggio perché si adatta e non deve mai risultare un malus), tieBreak per scontri mai giocati tra ELO vicini (prossimità lineare su 60 punti, peso maggiore al vertice, x12), malus abitudine x6, attività, equilibrio. I ruoli NON vincolano mai: due attaccanti e due difensori si possono suggerire e registrare normalmente.
- **Qualifica classifica (`QUAL`)**: 10 partite, 6 avversari diversi, 3 compagni diversi nella stagione. Sotto soglia: sezione "Non classificati" con ~ELO e progresso.
- **Fine stagione a scaglioni (`seasonStartElo`)**: 1250+ -> 1150, 1150+ -> 1100, 1050+ -> 1050, 950+ -> 1000, 850+ -> 950, 750+ -> 900, sotto -> 850. Da simulazione: il top con ~150 partite/stagione converge attorno a 1285 (range 1120-1450), regime in 40-60 partite, durata stagione consigliata ~3 mesi. Eventuale scaglione "1400+ -> 1200" da valutare solo coi dati veri.

## Mappa delle feature per tab

- **Classifica**: qualificati ordinati per ELO, non classificati con progresso, forma (ultimi 5), strisce, migliori coppie (top 5, min 3 insieme), striscia cliccabile per tornei con iscrizioni aperte. Tap su una riga apre il **profilo**: carica lo storico COMPLETO on-demand (query `.or(team_a.cs/team_b.cs)`), curva ELO in SVG, badge (cappotti, ammazzagiganti, strisce, titoli, cucchiai di legno, tornei), nemesi/vittima/compagno d'oro, testa a testa con select, ultime 20 partite.
- **Partita**: 1v1/2v2 (2v2 preselezionato), punteggio con griglia 0-10 a tap singolo, modalità "solo chi ha vinto", previsione con delta stimati, "Chi c'è oggi" con chip presenti e tavoli suggeriti. Admin registra direttamente, anonimo propone (pending).
- **Storico**: pending in cima con conferma/rifiuta (l'ELO si calcola alla CONFERMA coi rating correnti, il repetitionFactor sul created_at), filtro per stagione, undo dell'ultima confermata (i delta interi si sottraggono esattamente), badge rosso in nav. Il vincitore ha il trofeo davanti al nome e il perdente è attenuato (classe `.loser`). Gli admin possono **correggere qualsiasi partita** (matita): formazioni, punteggio, passaggio con/senza punteggio, vincitore, oppure **eliminarla**. Per le pending si aggiorna/cancella e basta; per le confermate della stagione attiva ogni salvataggio o eliminazione lancia automaticamente il ricalcolo completo della stagione. Le partite di stagioni chiuse non si toccano. NOTA: nel pannello di modifica i punteggi digitati vanno letti dal DOM e copiati nello stato PRIMA di chiamare withBusy, perché il render dello stato busy ricrea gli input.
- **Tornei**: eliminazione (seeding per rating coppia, 1ª vs ultima, bye alle migliori) o girone all'italiana; rated o "solo per la gloria"; modalità "iscrizioni aperte" (status `signup`: chiunque occupa un posto libero con conferma immutabile, admin gestisce tutto via select e avvia quando pieno); eliminazione torneo (le partite restano); albo tornei. Le partite rated dei tornei sono partite confermate normali.
- **Stagioni**: crea/pianifica (con ends_at facoltativa e countdown nell'header), coda numerata in ordine di attivazione, chiusura = premi in `awards` + snapshot in `season_results` + reset a scaglioni + avvio automatico della pianificata più vecchia, rinomina, **ricalcolo stagione** (rigioca TUTTE le confermate in ordine di confirmed_at; punto di partenza = `pre` della prima partita di ciascuno; riscrive deltas/pre/winners e i rating; idempotente; usa i boost salvati).
- **Giocatori**: rosa alfabetica (localeCompare it), totali di sempre dalla vista, proposta giocatori da anonimi con conferma admin, rinomina (aggiorna anche season_results), ruolo, login admin, backup JSON (fetch paginato di tutte le tabelle).
- **ELO**: scheda esplicativa completa delle regole, in italiano, per i giocatori.

## Dettagli implementativi da non dimenticare

- `state.matches` è una **finestra delle ultime 600 confermate** (per performance). Le statistiche di lungo periodo (profilo, premi, ricalcolo) fanno query dedicate complete. La vista `player_totals` esiste per lo stesso motivo.
- `boost` legacy: leggere sempre con `m.boost ?? (m.boosted ? 1.25 : 1)`.
- Realtime: canale su tutti i cambiamenti postgres, ricarica con debounce 500ms, saltata se `state.busy`.
- `withBusy` serializza le azioni di scrittura: blocca l'UI, esegue, fa `loadAll`, rirenderizza.
- I `deltas` sono interi arrotondati: l'undo li sottrae e ripristina esattamente.
- Le partite pending vengono valutate alla conferma coi rating del momento (scelta voluta), non con quelli dell'invio.
- La chiusura stagione NON è transazionale (chiamate in sequenza dal client): se fallisce a metà, controllare il pannello Supabase.

## Metodo di lavoro con Paolo

- Le modifiche all'HTML si fanno con **script Python di patch**: ogni `replace` preceduto da `assert count == 1` sulla stringa esatta, così una stringa non trovata o duplicata fa fallire la patch invece di corrompere il file in silenzio.
- Dopo ogni modifica, **verificare la sintassi del modulo JS con Node**: estrarre lo script, sostituire l'import di supabase con uno stub, `node --input-type=module --check`.
- Consegnare sempre il file completo in `/mnt/user-data/outputs/` e presentarlo.
- Se Paolo incolla un index.html "da cui partire", quello è la fonte di verità: confrontarlo con le ultime feature note, segnalare eventuali regressioni e chiedere/riapplicare.
