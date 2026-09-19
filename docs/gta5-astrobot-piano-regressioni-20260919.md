# GTA V / Astro Bot — indagine Fran1 → Fran2 e piano di implementazione

Data: 19 settembre 2026. Documento destinato all'agente che implementerà le correzioni.

## 1. Obiettivo e limiti dell'indagine

Recuperare stabilità, font e prestazioni di GTA V senza perdere i progressi di Astro Bot. L'utente riferisce che Astro Bot ora parte e raggiunge circa 12 FPS; GTA V crasha entrando nella storia, presenta font corrotti e scende a circa 3 FPS nelle impostazioni.

L'indagine confronta i sorgenti delle due directory, la storia Git e i log forniti. Non sono state eseguite nuove sessioni dei giochi: i 12 e 3 FPS sono osservazioni dell'utente, non misure ricavate dai log. Le cause visive e prestazionali richiedono ancora catture/profilazione; il punto di arresto della storia è invece identificato. Non attribuire ogni cambiamento a una correzione per Astro Bot: l'intervallo contiene anche refactoring e nuove funzionalità upstream.

**Priorità:** tessellazione per il crash; proprietà/coerenza dell'atlas R8 per i font; attese R8, DCC e readback per gli FPS. Non ripristinare in blocco il renderer Fran1.

## 2. Snapshot riproducibili e aggiornamento upstream

| Elemento | Identificazione |
|---|---|
| Sorgenti Fran1 | `W:/KytyLab/KytyPS5/Fran/KytyPS5-Fran1` |
| Sorgenti Fran2 | `W:/KytyLab/KytyPS5/Fran/KytyPS5-Fran` |
| Log Fran1 | `W:/KytyLab/KytyPS5/Fran/Build-Fran1/gta5-log.txt` |
| Log Fran2 | `W:/KytyLab/KytyPS5/Fran/Build-Fran/gta5-log.txt` |
| Build indicata dal log Fran1 | `9e2cdc0-dev` |
| HEAD della directory Fran1 | `1b0f159`, base `9e2cdc0` più skip dei draw patch |
| Build indicata dal log Fran2 | `21ee90e` |
| HEAD Fran2 all'inizio | `a5379c8`, con 9 file modificati |
| Verifica sostanziale | Prima dell'aggiornamento, `git diff 21ee90e --name-only` era vuoto: il worktree tracciato coincideva con la build del log |
| Upstream recuperato in questa sessione | `f100f78` |
| Merge locale su main | `9e515a3` |
| Backup pre-merge | branch `backup/main-before-upstream-20260919` |
| Copia delle modifiche originarie | stash con messaggio `Preserve GTA5 local changes before upstream 20260919`, mantenuto dopo l'applicazione |

I tre commit upstream integrati sono `ea6aae8` (V_MAD_I16), `ba1854b` (ID pool rete da 1), `f100f78` (output stato controller/broadcast). Merge e riapplicazione delle modifiche locali senza conflitti. Questi tre cambiamenti non correggono direttamente la tessellazione o il percorso dell'atlas identificati qui. Nessun push e nessuna installazione sopra `Build-Fran` sono necessari per usare questo piano.

Il log Astro Bot già disponibile in `Build-Fran/astrobot-log.txt` indica `58e4984`, titolo `PPSA21567`, versione `01.018.000`; termina con chiusura finestra e salvataggio cache. Quindi non usare automaticamente la vecchia diagnosi di arresto contenuta in `docs/astrobot-handoff.md` come descrizione dello stato attuale. Quel documento resta utile per le invarianti SRT/dispatcher da preservare.

### Modifiche locali: revisione richiesta dall'utente

Sono state valutate separatamente dalle differenze storiche:

- **R8 early-out:** conservare. `IsStorageSampledFormatMismatch` riconosce soltanto la coppia `k8UInt`/`k8UNorm`; evitare la ricerca preliminare per gli altri formati conserva il dominio del controllo. Era già presente nella build del log: non è una nuova soluzione ai 3 FPS.
- **Diagnostica R8:** conservare, con commenti che non dichiarino misure inesistenti. Otto messaggi sono il limite del contatore, non il totale degli eventi.
- **Fallback tessellazione → VS normale:** rimuovere. Fallisce prima del ritorno opzionale sul branch interno del log; inoltre `GetDrawTopology` lascia `ePatchList` mentre il fallback elimina HS/TES. Il chiamante può aver già validato header HS e stage attivi: stride sconosciuto non prova shader non tessellati. Ripristinare il comportamento esplicito di funzionalità non supportata è preferibile a registrare una falsa correzione. Questo può riportare il primo arresto a uno stride zero, prima del branch del log; NON è una soluzione della storia.
- **Store dinamici generalizzati:** sostituire l'applicazione a qualunque hash con una allowlist delle due coppie note, mantenendo anche tutti i controlli sulla forma dell'istruzione. Aggiungere test di esclusione per hash/PC non corrispondenti. È un workaround con perdita di scritture, non implementazione corretta del kernel.
- I cambiamenti alla suite tessellazione che ignoravano il ritorno di `TryAnalyzeTessellationPrograms` non costituiscono un test del fallback; rimuoverli insieme all'API opzionale.

L'esito finale di build/test e i commit della revisione sono riportati nella sezione di verifica in fondo.

## 3. Evidenze dai log

Entrambe le sessioni GTA usano `PPSA04263`, versione `01.010.002`, RTX 5080, Ryzen 9 9950X3D, finestra iniziale 1280×720. I commenti del workaround menzionano anche `PPSA04264`: non confondere i due title ID.

| Evidenza | Fran2 | Fran1 | Interpretazione |
|---|---|---|---|
| Righe totali | 337.220 | 4.653.893 | Durate e verbosità differenti: non confrontare conteggi grezzi come prestazioni |
| Fine sessione | Fatal Error | Window closed / quit | Arresto esplicito vs chiusura richiesta |
| Cache pipeline | inizializzata vuota | disabilitata, dirty build | Separare prima compilazione e stato caldo |
| Campioni FPS | nessuno | nessuno | Profilazione da aggiungere |
| Messaggi R8 | 8, stesso indirizzo `0x25b491000` | assenti | Fran1 non ha quel log: assenza non implica assenza del percorso |
| Fallback stride LS | 3 | 0 | La correzione provvisoria era già attiva e insufficiente |
| Skip patch | 0 | 8 | Fran1 non renderizza quei draw |
| Messaggi `CFG dispatcher` | 0 | 0 | Nessuna evidenza che il dispatcher spieghi il rallentamento GTA osservato |
| `VUID-` / errore fence GPU | nessuno osservato | nessuno osservato | Non prova validazione abilitata o correttezza GPU |

Riferimenti nel log Fran2:

- Riga 78.005: prima pubblicazione dell'atlas R8; altre alle righe 78.012–78.013 e successive fino al tetto di otto messaggi.
- Righe 334.457, 334.482, 334.530: `Tessellation stride analysis failed for LS, falling back to plain vertex path`.
- Righe 337.219–337.220: `Fatal Error`, `IsDirectBranch(inst.opcode) && inst.branch_target != program.instructions.back().pc`, `Tessellation.cpp:87`.
- Gli ultimi VS/PS stampati non identificano LS/HS responsabile del crash: occorre registrare gli shader tessellati prima dell'analisi.

Riferimenti Fran1: righe 341.035–343.795 contengono gli otto `skipping patch-topology draw without tessellation support`. Questo rende compatibile il log `9e2cdc0-dev` con la patch poi committata in `1b0f159`, ma la stringa versione da sola non certifica tutti gli input di build.

Sono presenti 250 coppie uniche stage/hash con emissione SPIR-V in Fran2 e 526 in Fran1; 244 sono comuni. Alcuni shader VS piccoli crescono: `0x6c02baa8b92efb43` passa da 618 a 904 word. Non convertire questa crescita in una stima di FPS: mancano frequenze d'uso, permutazioni e tempi GPU; i due run percorrono scene differenti.

## 4. Crash entrando nella storia: causa accertata e soluzione

### Catena verificata

`PipelineCache::GetGraphicsPrograms` seleziona la tessellazione su `kPatch` → `PrepareTessellationPrograms` prepara LS/HS/TES → `ReflectTessellationStride` visita linearmente le istruzioni e interrompe il processo su un branch diretto interno. Il log termina esattamente su questo controllo.

La funzionalità LS/HS/TES arriva con `7516068`; `87971a2` isola l'analisi in `Tessellation.cpp`. Il successivo tentativo `f50f116` gestisce solo stride zero e non quel branch. Fran1 non dispone dello stesso percorso, e la sua patch locale evita il problema saltando i draw.

### Attività T1 — catturare il caso reale

File: `src/graphics/shader/shader.cpp`, `recompiler/Tessellation.cpp`, `host_gpu/renderer/pipeline/pipelineCache.cpp`.

1. Prima di `AnalyzeTessellationPrograms`, registrare hash e tipo header di LS/HS/TES, indirizzi guest, stage register, topologia, control point count, PC/target del ramo non gestito.
2. Salvare una fixture circoscritta di codice shader e stato rilevante; non dedurre LS/HS dagli ultimi log VS/PS.
3. Distinguere: vera tessellazione con flusso complesso, lettura errata dei registri/header, draw non tessellato con stato incoerente. I commenti che affermano «plain shaders» non sono evidenza.

### Attività T2 — rendere l'analisi affine sensibile al CFG

File: `Tessellation.cpp/.h`; riutilizzare decoder/CFG già presenti, senza duplicarne la decodifica.

- Propagare uno stato astratto dei registri per basic block con worklist. Distinguere blocco non raggiunto da valore sconosciuto; usare gli stati affine/coefficient/constant e packed-control-point esistenti.
- Al join conservare solo valori uguali su tutti i predecessori raggiungibili; conflitti diventano unknown. Per loop, convergenza finita tramite degradazione a unknown, senza inseguire indefinitamente coefficienti crescenti.
- Trattare le uscite finali senza inventare un arco eseguito; non linearizzare entrambi i rami come se fossero in sequenza.
- Verificare ogni indirizzo LDS/ring effettivamente rilevante: stride coerenti e allineati, control point validi, zero/overflow, LS e HS indipendenti. Ampliare le operazioni affine solo quando una fixture le richiede.
- Restituire una diagnosi strutturata per i casi non dimostrabili, comprensiva di stage/hash/PC/motivo. La mancata dimostrazione non autorizza VS normale, stride arbitrario o soppressione silenziosa dei draw.
- Se serve un workaround transitorio per continuare l'indagine, deve essere esplicito, circoscritto alla fixture e classificato come resa incompleta. Non promuoverlo a correzione finale.

Test richiesti: branch interno con definizioni uguali e differenti; ramo che esce; loop con stride invariato; indirizzo non affine; stride zero; numero control point invalido; risultati di LS/HS/TES e pixel/geometry output su GPU per una piccola patch nota. I test attuali `CheckTessellationPrograms` coprono arithmetic pattern, non questa matrice.

**Accettazione:** GTA entra nella storia senza il fatal e senza introdurre nuovi skip; patch effettivamente disegnate; nessuna pipeline con topologia e stage incompatibili. Verificare anche `kRectList`, che usa tessellazione sintetica ed è un percorso distinto da `kPatch`.

## 5. Font: separare contenuto dell'atlas, campionamento e compositing

### F1 — proprietà R8 e alternanza storage/sampled, priorità alta

File: `textureCache.cpp` (`PrepareStorageSampledOverlap`, `FindImage`, `InitializeImage`, `CommitGpuWrite`), `descriptors.cpp` (`ResolveTexture`, `BindImage`, `RebindImages`), `imageView.cpp`.

Il codice corrente riconosce l'alias R8 UINT/UNORM, scarica i dati GPU, chiama `Finish` e `WaitPriorityOperations`, invalida il buffer, elimina il vecchio owner e ricrea l'immagine nel formato richiesto. Il nucleo di questo percorso esisteva già in Fran1: **non è corretto attribuire la regressione alla sola sua introduzione**. È cambiato il contesto di invalidazione, risoluzione alias, trasferimento e binding.

Esperimento: contatori per indirizzo/frame su transizioni UINT→UNORM e inverse, creazioni/delezioni, byte trasferiti, dirty state e attese. Limitare il dump al range dell'atlas catturato, non hardcodificare `0x25b491000` nel prodotto. Confrontare la stessa schermata Fran1/Fran2.

Catturare: immagine subito dopo il producer, backing dopo il download, immagine campionata e risultato del draw. Usare glyph non uniformi e canali alfa intermedi. Se il producer è già errato, indagare shader/descriptor; se si corrompe nel passaggio owner, indagare cache/trasferimento; se l'atlas è corretto ma il testo no, passare a F3.

Soluzione preferita da dimostrare: backing GPU unico con viste UINT/UNORM quando creazione/formati/usage lo consentono, dirty generation unica e dipendenze compute→fragment corrette. Se non è applicabile, copia/trasferimento GPU con preservazione dei byte e versioni degli owner; mantenere il percorso CPU come fallback per gli altri casi. Non basta cancellare `Finish` o tenere due owner senza sincronizzarli.

Test: alternare più frame storage write→sample→storage partial write→sample; modifiche CPU tra due usi; dirty buffer che aliasa il medesimo range; atlas tiled 2048×4096 e caso lineare; nessun riutilizzo di una vista eliminata; valori R8 letti come UNORM e alfa attesi. Il test esistente con clear uniforme `0x35` non individua permutazioni dei texel o tutti gli errori di aggiornamento parziale.

### F2 — invalidazione buffer→immagine e alias stencil, priorità alta

In `NativeStorageBuffer`, Fran1 invalidava le immagini solo per `resource.formatted && resource.written`; Fran2 lo fa per ogni buffer scritto. `InvalidateMemoryFromGPU` cancella `IsGpuModified` e marca `BufferModified`. Il principio può essere necessario per vere scritture alias, ma richiede che le parti non sovrascritte siano prima preservate e che il nuovo owner abbia dati aggiornati.

Strumentare range reali e range del descrittore: una dichiarazione writable non dimostra una scrittura a ogni byte. Testare scrittura parziale del buffer sopra un'immagine GPU dirty e campionamento successivo delle parti intatte. Controllare l'ordine `FindBuffers`/`RebindImages`/`RebindBuffers` introdotto o riorganizzato da `dd408ff` e `6d1ba58`; non ripristinare alla cieca il vecchio filtro `formatted`, che perderebbe scritture valide.

`AssociateStencil` e il riuso R8 meritano un controllo dedicato: `47957fa` ha già introdotto matching per range ed extent. I test su riuso di una superficie R8 come stencil sono presenti. Verificare che il caso reale dell'atlas non venga associato a uno stencil soltanto per indirizzo compatibile.

### F3 — se i byte dell'atlas sono corretti

Esaminare `TextureViewInfo`, `TextureGetComponentMapping`, `spirvEmitterImage.cpp`, PS export, maschere e blending:

- `a305a6c`: layout canali condiviso texture/render target, `host_to_storage`; verificare R001 e 000R, alfa e assenza di doppio swizzle tra shader e view.
- `6493674`: dual-source blending; verificare i fattori effettivi del draw dei font e le location/index degli export. La rimozione di `blend_bypass` da una struttura non implica che sia perso: la condizione è stata trasferita in `pipelineCache.cpp`.
- Nuovi slot MRT non compattati e stencil dinamico: verificare target slot, write mask, stencil state quando si torna al menu o cambia passata.
- `f38d738`: MinLOD e `ed3b3cf`: border color table. Catturare descrittore e sampler, confrontare valori e chiavi cache. LOD è meno sospetto per un atlas a un solo mip; non impostarlo globalmente a zero.

`libFont.cpp` cambia accettazione OTTO e API fallback VGA FFmpeg; il log mostra caricamenti di font `.gfx`, non una prova di rasterizzazione tramite libFont. Anche i font del launcher Qt sono separati dai font renderizzati dentro GTA.

### Controllo negativo eseguito sul tiler

`gpu_tiler_standard64.inc` è stato rifattorizzato usando `standard_offset`. Le espressioni vecchie e nuove sono state confrontate per 327.680 combinazioni: x/y in 0..255, casi 1/2/4/8/16 byte; identiche. Non dare priorità a un rollback di quelle formule. Restano da verificare descrittori, layout mip, pitch e regioni di copia: l'uguaglianza degli offset non li copre.

## 6. Prestazioni: individuare le attese, poi ottimizzare

### P0 — baseline misurabile

Registrare identità eseguibile e sorgenti, GPU/driver, opzioni CLI effettive, present mode, vblank, validazione, debug dump, shader optimization, readback e stato cache. I default present mode sono passati da Fifo a Mailbox; il launcher passa l'opzione esplicita, quindi leggere gli argomenti reali.

Usare stesso salvataggio, percorso, risoluzione e camera: menu iniziale, pagina impostazioni specifica, ingresso nella storia, breve sequenza stabile; Astro Bot stessa sequenza fino alla scena giocabile. Separare run freddo e caldo. Tre campioni di almeno 30 secondi per schermata stabile; frame time mediano e p95, CPU submit, tempi GPU e tempo atteso dalla CPU. Non ricavare FPS dalla dimensione del log o dai clock EndOfPipe.

Diagnostica e benchmark sono run distinti: verificare correttezza con validazione e poi misurare entrambi con identiche impostazioni. Non disabilitare controlli per nascondere un errore.

### P1 — R8

Misurare durata e frequenza di `PrepareStorageSampledOverlap`, `DownloadImageMemory`, `Finish`, `WaitPriorityOperations`; byte e creazioni immagini per frame. Otto righe non dimostrano un evento a frame e non escludono migliaia di eventi dopo il limite del log. Se domina, implementare F1. Il filtro dei formati estranei è utile ma era già nella build lenta.

### P2 — DCC e readback, candidato distinto e importante

Commit `01df42a`, `437e69e`, `ea092a9`; `MaterializeDccClear` ha sostituito il tracking precedente. Ora controlla dirty metadata, può chiamare `BufferCache::ReadMemory`, legge backing, alloca/scansiona una slice uniforme, emette clear e consuma la chiave con `FillBuffer`.

Questo può incidere anche sulle impostazioni prive di producer dell'atlas. Misurare per frame: chiamate, metadata già consumati, byte scansiti, allocazioni, readback retired/fallback, attese e clear. Distinguere DCC da R8: la funzione di mismatch R8 esclude immagini con metadata.

Ottimizzare con generazioni/invalidation dei metadata e caching del risultato solo quando invariato; validare cambio dei clear-register per il codice `0x20`, layer/mip/view format, riuso DCC↔HTile, scritture CPU e GPU, compute che non scrive un clear uniforme. Non ripristinare un flag «clear una volta per indirizzo» senza seguire le nuove scritture.

### P3 — readback retired e staging

Commit `283c293`, `23c8a43`, integrazione `330e415`; file `bufferCache.cpp`, tracker e scheduler. Il percorso retired evita il drain del tick corrente ma usa comunque submit/fence sincrona; «retired» non significa costo nullo. Esiste un fallback con attesa del tick corrente. La finestra di 512 KiB era già in Fran1: non presentarla come nuova causa.

Contare fault CPU, tick richiesto/completato, cause del fallback, numero intervalli nella mappa dei write tick, byte richiesti/scaricati, submit separati e tempo fence. Migliorare batching e coalescenza per la workload reale, preservando l'ultimo writer di ogni intervallo. Testare riscrittura successiva, intervalli parziali, read-modify-write CPU, GC e unmap. I dati vanno pubblicati prima di togliere tracking/protezioni o rilasciare owner.

### P4 — SRT/BDA, risorse e shader

L'evoluzione SRT mantiene letture dinamiche e condizionate sulla GPU per Astro Bot. Verificare quanti shader GTA attivano `uses_dma`, costo di `PrepareBda` e `SynchronizeBuffersInRange`; confrontare l'elenco prima/dopo, non assumere che ogni accesso BDA sia nuovo. Una lettura anticipata CPU di un indirizzo non valido resta sbagliata anche se più veloce.

`LowerDescriptorPhi` (`af3a3f4`) consente selezioni uniformi e senza scritture; mantenere tali precondizioni. Profilare materializzazione e chiavi/permutazioni programma; distinguere crescita di shader da ricompilazioni ripetute. Il log GTA non mostra selezioni del dispatcher: non farne il primo bersaglio senza nuove evidenze.

### P5 — cause secondarie

Logging, nuova cache sampler (lettura tabella prima del lookup), primitive/MRT/depth discovery, present/vblank, kernel/threading, audio, allocazioni e migrazione memoria. Prima attribuire il tempo a CPU, GPU, attesa o presentazione; poi usare bisect mirati. Nel log Fran1 ci sono molti più messaggi di command packet: la mera verbosità maggiore non spiega Fran2 più lenta.

## 7. Store dinamici: debito da eliminare senza danneggiare Astro Bot

`TryDisableDynamicBufferStore` della build del log azzera il descrittore e rende falso il predicato dello store. La forma dell'istruzione non prova che sia lecito perdere la scrittura. La generalizzazione di `21ee90e` può interessare altri shader/giochi e non produce un log per applicazione: dai log forniti non si ricava quante scritture siano state soppresse.

La revisione locale limita il comportamento alle coppie già documentate `(0x6a53456e7ef5d1b0, 0x86c)` e `(0xf1e6128a7eecc3c4, 0x98c)` più shape guard. Non estendere l'allowlist per ogni crash successivo. L'uso della seconda coppia proviene dalla diagnosi codificata nelle modifiche esistenti, non da una nuova cattura effettuata qui.

Soluzione definitiva: implementare store tramite descrittore dinamico per-lane rispettando formato, stride, bounds/OOB, EXEC e indirizzo, con un meccanismo di tracciamento delle pagine/range scritti e loro ownership. Integrare visibilità verso buffer, texture alias e letture CPU. Non basta emettere una scrittura BDA libera senza tracking.

Test numerici: lane che selezionano destinazioni differenti, predicato falso, pagina non mappata gestita secondo contratto, overflow/OOB, readback CPU e sampling di alias dopo lo store. Ridurre/eliminare i workaround solo dopo equivalenza su questi test e sul kernel reale.

## 8. Cosa preservare per Astro Bot

- Ray/BVH introdotto dall'integrazione `589ae76`, emitter e accessi BDA necessari. Non disattivare ray tracing per recuperare FPS in GTA.
- Piano SRT di `79f58cf` e successive integrazioni: `ValidateRuntimeValue`, niente dereference anticipato di pointer chase/letture condizionate, dynamic descriptor tuple su GPU.
- Dispatcher per CFG non emettibili e correzioni Phi: snapshot simultaneo delle sorgenti, aggiornamento soltanto dell'arco selezionato, gestione wave32/wave64.
- Scope delle barriere per stage e guardie OOB coerenti con la nuova API SPIR-V (`969587f`).
- Retired readback e pubblicazione dei dati senza duplicare il download; nessuna rimozione globale delle sincronizzazioni.
- Mip tail e promozione delle immagini (`35cf2f8`, `bbebb64`, `8589731`), preservazione dei livelli già scritti GPU. Un fix dell'atlas single-mip non deve restringere globalmente i descriptor multi-mip.
- Proprietà e tracking per memoria guest, shader mesh/NGG, depth/stencil e clear reali.

Queste sono invarianti da testare, non una prova che ogni implementazione corrente sia corretta. Preferire selezione del comportamento in base a stage, layout e ownership; limitare eccezioni per titolo/shader ai workaround documentati, temporanei e misurabili.

## 9. Sequenza operativa per il prossimo agente

**Prerequisito emerso dai test della sessione:** la base aggiornata ha cinque CTest rossi, riprodotti anche togliendo le modifiche locali in revisione. Prima di considerare protette le invarianti Astro/GTA, classificare i tre difetti sottostanti:

- `shader_cfg`: `vertex barrier does not use subgroup execution scope`. Il test cerca letteralmente `OpControlBarrier %uint_3 %uint_3`; l'emitter attuale usa execution scope Subgroup ma memory scope Workgroup nei vertex shader. Analizzare legalità e semantica dei due scope e delle memory semantics separatamente; non cambiare semplicemente la stringa attesa per ottenere verde.
- `resource_tracking`: `conditional sampler phi: nonuniform sampler selection was accepted`. `CheckFatal` usa quel messaggio sia quando non arriva l'eccezione sia quando il suo testo è diverso da quello atteso. Registrare il motivo originale e distinguere un test di diagnostica obsoleto dall'accettazione reale di un predicato non uniforme. Non disattivare la verifica di uniformità.
- `shader_recompiler_compute`, `graphics_pipeline_rasterization`, `texture_cache_image_overlap`: medesimo punto `PolygonModeRasterization / NULL-export full depth clear`, con messaggio su color target non esportato. Il setup del test ha stencil attivo e depth write disabilitato; il codice corrente conserva l'HTile clear nei passaggi depth read-only. Distinguere aspettativa di clear da riduzione della render area dovuta a MRT obsoleti; il messaggio da solo non dimostra quale sia errato. Leggere i pixel effettivi e verificare clear, stencil coverage e discard separatamente.

Questi risultati sono evidenze aggiuntive utili per F3/HTile e shader; non provano da soli la causa dei font o dei 3 FPS. Il fallimento anticipato delle suite impedisce ad alcuni loro casi successivi di eseguire: 37 CTest verdi non equivale a tutti i sottotest rimanenti verificati.

1. **Congelare baseline e configurazioni.** Usare worktree/build separate per Fran1, `21ee90e` e main aggiornato; preservare log e cache originali. Non usare `reset --hard` sulla directory principale e non applicare di nuovo lo stash conservato.
2. **Riprodurre crash e aggiungere cattura T1.** Acquisire anche i primi stride zero, perché il fallback insicuro è stato rimosso. Fixture del vero LS/HS, non solo test inventato.
3. **Implementare T2 in un commit autonomo.** Test shader e pipeline + storia GTA. Nessun lavoro di performance nello stesso commit.
4. **Catturare atlas e draw dei font.** Implementare F1/F2 solo dopo localizzazione del primo dato errato; passare a F3 se il contenuto è corretto.
5. **Profilare P0–P4.** Inserire contatori aggregati o tracing circoscritto. Scegliere l'intervento dal tempo misurato; un cambiamento per esperimento.
6. **Implementare ottimizzazioni mantenendo le invarianti.** Preferire percorso comune corretto; A/B temporanei non devono diventare interruttori permanenti che saltano lavoro.
7. **Affrontare store dinamici reali.** Eliminare progressivamente il workaround con equivalenza numerica; segnalarne esplicitamente l'eventuale permanenza nel resoconto.
8. **Eseguire matrice completa e documentare.** Commit separati, hash binari, log/catture/risultati, problemi residui e istruzioni riproducibili.

Se serve una bisezione storica: confrontare le integrazioni `589ae76`, `9678765`, `79f58cf`, `e2ad0b4`, `58e4984`, poi i gruppi funzionali indicati nelle sezioni precedenti. Alcuni punti intermedi richiedono gli adattamenti di build successivi: non scambiare un errore di compilazione per una regressione del gioco. Non copiare interi file Fran1 sopra le API nuove.

### Matrice minima di accettazione

| Area | Verifica | Esito richiesto |
|---|---|---|
| GTA menu/impostazioni | stessi schermate e font, run caldo | pixel/font corretti; frame time confrontabile con Fran1; obiettivo iniziale entro 10% della baseline calda misurata |
| GTA storia | caricamento e sequenza giocabile ripetuta | niente abort tessellazione, niente nuovi skip o write suppression |
| Atlas | pattern non uniforme, transizioni ripetute | texel/alfa corretti, nessun readback completo ricorrente quando evitabile |
| Astro Bot | boot e stessa scena di riferimento | progressione preservata; FPS entro variabilità concordata rispetto al riferimento circa 12, misurato nelle stesse condizioni |
| GPU correctness | validazione e risultati numerici | niente nuovi errori, nessun device lost, nessuna perdita silenziosa di dati |
| Cache/lifetime | frame multipli, GC e riuso indirizzi | owner/dirty state coerenti, niente vista obsoleta |

I target prestazionali sono criteri proposti, non risultati ottenuti in questa indagine.

## 10. Verifica automatica e comandi

Ambiente locale: Windows, clang-cl, Visual Studio 18 Community, Qt 6.10.3, configurazione Release in `_Build/windows`. Gli helper impostano l'ambiente; `build-kyty.bat` include installazione e pausa interattiva, quindi non usarlo alla cieca per le sole verifiche.

```powershell
cmd /c _Build\build-upstream.cmd
cmd /c _Build\test-upstream.cmd
git diff --check
```

Target/suite rilevanti già esistenti: `resource_tracking`, `resource_materialization`, `shader_cfg`, `shader_recompiler_compute`, `shader_bvh_decode`, `shader_bvh_intersection`, `graphics_pipeline_rasterization`, `graphics_draw_offsets`, `gpu_tiler`, `texture_cache_storage_sampled`, `texture_cache_image_views`, `texture_cache_image_overlap`, `texture_cache_layered_image`, `texture_cache_htile_clear`, `texture_cache_depth_readback`, `buffer_cache_ranges`, `buffer_cache_dirty_gc`, `command_scheduler_timeline`, `stream_buffer_ring`, `pm4_context_state`, `memory_tracker`, `virtual_memory_allocation`, `library_ui_tests`. Consultare `CMakeLists.txt`/`ctest -N` per disponibilità e nomi effettivi sulla piattaforma.

Non assumere «40 test verdi» dal vecchio handoff: contare la suite attuale. La sola compilazione SPIR-V non dimostra correttezza numerica, e CTest verde non certifica i due giochi.

## 11. Copertura delle modifiche e aree secondarie

L'inventario in appendice è generato da `git diff --numstat 9e2cdc0 21ee90e` e copre 268 percorsi tracciati. Il confronto fisico `src` tra le directory iniziali rileva 243 file, più 10 file di test; Fran1 ha in più lo skip patch di `1b0f159`. L'inventario non è una dichiarazione che ogni riga sia stata provata in esecuzione o che tutti i file siano colpevoli.

| Gruppo | Valutazione / azione |
|---|---|
| Tessellazione, pipeline, draw | Crash accertato nel nuovo percorso; T1/T2 |
| Texture, buffer cache, descriptor, sync, tiler | Principali candidati font/FPS; F1/F2/P1–P3 |
| Decoder, IR, SPIR-V, SRT, CFG | Proteggere Astro; test numerici e P4; verificare shader font solo dopo cattura |
| Depth/HTile/DCC, MRT/blend/sampler | F2/F3/P2; non confondere refactoring con perdita di semantica |
| Memoria guest, page/region manager, fault, loader | Possibili costi/corruzioni indirette; priorità se profiling o cattura indica fault/range errati |
| Kernel, pthread, eventFlag, syncOnAddress | Possibili attese CPU; non spiegano il fatal esplicito tessellazione |
| Audio/AJM/AV, rete, controller, servizi | Controllare solo in presenza di tempi CPU/anomalie correlate; nessun nesso diretto dimostrato con atlas |
| Launcher/QML/font Qt/config | Possibili differenze delle opzioni lanciate; non correggono l'atlas del gioco |
| Dipendenze/build/CI/versione/documenti | Riproducibilità e build; verificare versioni nel confronto, nessuna attribuzione automatica del calo FPS |

## 12. Esito della sessione

- Upstream `f100f78` integrato su main con merge `9e515a3`; modifiche originarie salvate prima del merge e poi riviste. Lo stash è conservato come recupero, non come lavoro da riapplicare.
- Build Release di emulatore, launcher e target dell'helper riuscita. È stato corretto anche il test console `KernelFileSystemTests.cpp`: `SDL_MAIN_HANDLED` impedisce che SDL rinomini la sua funzione `main`, causa del precedente errore di link. Log finale build: `_Build/upstream-20260919-reviewed-build.log`.
- Prima suite con modifiche riviste: **37/42 CTest passati**, cinque fallimenti sopra, 32,03 secondi. Controprova sulla base `9e515a3` senza i cambiamenti R8/store/test-store, conservando soltanto il fix del link filesystem: **gli stessi cinque fallimenti**, 29,55 secondi. Log `_Build/upstream-20260919-tests.log` e `_Build/upstream-20260919-baseline-tests.log`.
- Test mirato aggiunto perché la suite resource tracking si arresta prima del caso GTA: `_Build/windows/resource_tracking_tests.exe --gta-dynamic-store-only` **passa**. Verifica entrambe le coppie note, hash estraneo, PC estraneo e coppie incrociate. Il selettore non modifica il comportamento predefinito della suite.
- Test finali dopo ripristino della revisione: **37/42 passati**, gli stessi cinque fallimenti, 29,03 secondi; `_Build/upstream-20260919-reviewed-tests.log`.
- Commit locali della revisione: `3dc38ff` (entry point test filesystem) e `7adaa91` (allowlist store, test mirati, filtro/diagnostica R8). Il fallback tessellazione originariamente non committato è stato scartato, non trasferito nel commit. Upstream e revisione sono locali: nessun push eseguito.
- `git diff --check` senza errori. Confronto delle formule Standard64 superato per 327.680 coordinate/casi.
- Nessuna nuova esecuzione GTA/Astro, nessun risultato FPS dichiarato, nessuna installazione nei due Build di riferimento. Crash storia e correzione definitiva dei font/FPS rimangono lavoro del piano.

## Appendice A — inventario completo baseline Fran1 → build Fran2 del log

Formato: righe aggiunte, righe rimosse, percorso. Non comprende i tre commit upstream del 19 settembre né le correzioni di revisione eseguite dopo l'indagine.

```text
37	0	.coderabbit.yaml
21	0	.github/workflows/build.yml
1	1	.gitmodules
6	4	.vscode/settings.json
27	15	CMakeLists.txt
35	4	README.md
423	0	docs/astrobot-handoff.md
56	0	docs/upstream-integration-20260909.md
27	0	flake.lock
20	0	flake.nix
84	0	nix/devshell.nix
22	0	patches/README.md
15	0	patches/spirv-tools-vscode-activation.patch
4	0	shell.nix
25	0	src/common/alignment.h
2	18	src/common/assert.cpp
0	166	src/common/byteBuffer.h
0	17	src/common/debug.cpp
1	27	src/common/debug.h
16	4	src/common/emulatorConfig.cpp
14	12	src/common/emulatorConfig.h
13	28	src/common/file.cpp
4	7	src/common/file.h
0	97	src/common/hash.h
11	3	src/common/logging/log.cpp
1	0	src/common/logging/log.h
0	42	src/common/magicEnum.h
1	3	src/common/platform/sysDbg.h
0	1	src/common/platform/sysFileIO.h
0	51	src/common/platform/sysLinuxDbg.cpp
1	4	src/common/platform/sysLinuxFileIO.cpp
37	106	src/common/platform/sysLinuxVirtual.cpp
0	53	src/common/platform/sysSwapByteOrder.h
1	5	src/common/platform/sysTimer.h
0	28	src/common/platform/sysVirtual.h
0	12	src/common/platform/sysWindowsDbg.cpp
3	6	src/common/platform/sysWindowsFileIO.cpp
31	49	src/common/platform/sysWindowsVirtual.cpp
7	13	src/common/profiler.cpp
2	180	src/common/stringUtils.h
8	51	src/common/threads.cpp
0	2	src/common/threads.h
0	17	src/common/timer.cpp
0	6	src/common/timer.h
0	63	src/common/virtualMemory.cpp
1	1	src/common/virtualMemory.h
4	0	src/emulator.cpp
5	5	src/graphics/guest_gpu/command_processor/commandProcessor.h
97	28	src/graphics/guest_gpu/command_processor/pm4Handlers.cpp
9	0	src/graphics/guest_gpu/gpu_defs.h
72	67	src/graphics/guest_gpu/graphicsRun.cpp
50	14	src/graphics/guest_gpu/hardwareContext.h
9	0	src/graphics/guest_gpu/pm4.h
17	30	src/graphics/guest_gpu/tile.cpp
1	0	src/graphics/guest_gpu/tile.h
5	23	src/graphics/host_gpu/graphicContext.h
26	0	src/graphics/host_gpu/hostMemory.cpp
2	0	src/graphics/host_gpu/hostMemory.h
3	5	src/graphics/host_gpu/memoryTracker.cpp
8	33	src/graphics/host_gpu/memoryTracker.h
15	64	src/graphics/host_gpu/pageManager.cpp
2	14	src/graphics/host_gpu/rangeSet.h
20	39	src/graphics/host_gpu/regionManager.h
303	126	src/graphics/host_gpu/renderer/cache/bufferCache.cpp
19	2	src/graphics/host_gpu/renderer/cache/bufferCache.h
1	6	src/graphics/host_gpu/renderer/cache/faultManager.cpp
0	134	src/graphics/host_gpu/renderer/cache/gpuResourceManager.cpp
0	52	src/graphics/host_gpu/renderer/cache/gpuResourceManager.h
51	15	src/graphics/host_gpu/renderer/cache/samplerCache.cpp
3	1	src/graphics/host_gpu/renderer/cache/samplerCache.h
15	45	src/graphics/host_gpu/renderer/cache/streamBuffer.cpp
9	7	src/graphics/host_gpu/renderer/cache/streamBuffer.h
217	242	src/graphics/host_gpu/renderer/cache/textureCache.cpp
34	25	src/graphics/host_gpu/renderer/cache/textureCache.h
9	15	src/graphics/host_gpu/renderer/colorRenderTarget.cpp
1	7	src/graphics/host_gpu/renderer/commandScheduler.cpp
0	1	src/graphics/host_gpu/renderer/commandScheduler.h
2	3	src/graphics/host_gpu/renderer/context.cpp
73	128	src/graphics/host_gpu/renderer/debug.cpp
1	4	src/graphics/host_gpu/renderer/debug.h
213	122	src/graphics/host_gpu/renderer/depthRenderTarget.cpp
2	5	src/graphics/host_gpu/renderer/depthRenderTarget.h
2	15	src/graphics/host_gpu/renderer/image/blitHelper.cpp
1	2	src/graphics/host_gpu/renderer/image/blitHelper.h
21	26	src/graphics/host_gpu/renderer/image/image.cpp
2	1	src/graphics/host_gpu/renderer/image/image.h
39	98	src/graphics/host_gpu/renderer/image/imageInfo.h
12	6	src/graphics/host_gpu/renderer/image/imageView.cpp
176	262	src/graphics/host_gpu/renderer/image/textureCommon.cpp
8	12	src/graphics/host_gpu/renderer/image/textureCommon.h
17	30	src/graphics/host_gpu/renderer/image/tiler.cpp
3	3	src/graphics/host_gpu/renderer/pipeline/descriptorHeap.cpp
1	1	src/graphics/host_gpu/renderer/pipeline/descriptorHeap.h
176	139	src/graphics/host_gpu/renderer/pipeline/descriptors.cpp
3	8	src/graphics/host_gpu/renderer/pipeline/descriptors.h
149	72	src/graphics/host_gpu/renderer/pipeline/pipelineCache.cpp
27	24	src/graphics/host_gpu/renderer/pipeline/pipelineCache.h
26	6	src/graphics/host_gpu/renderer/pipeline/shaderResourceBarrier.cpp
4	3	src/graphics/host_gpu/renderer/pipeline/shaderResourceBarrier.h
68	287	src/graphics/host_gpu/renderer/pipeline/shaders.cpp
11	15	src/graphics/host_gpu/renderer/render.h
28	46	src/graphics/host_gpu/renderer/renderCompute.cpp
84	3	src/graphics/host_gpu/renderer/renderContext.cpp
19	5	src/graphics/host_gpu/renderer/renderContext.h
241	266	src/graphics/host_gpu/renderer/renderDraw.cpp
3	3	src/graphics/host_gpu/renderer/renderDraw.h
0	36	src/graphics/host_gpu/renderer/renderTarget.h
49	122	src/graphics/host_gpu/renderer/sync.cpp
12	24	src/graphics/host_gpu/shaders/gpu_tiler_standard64.inc
18	41	src/graphics/host_gpu/vma.cpp
0	14	src/graphics/host_gpu/vma.h
27	1	src/graphics/host_gpu/vulkanCommon.cpp
2	3	src/graphics/host_gpu/vulkanCommon.h
14	19	src/graphics/presentation/videoOut.cpp
2	0	src/graphics/presentation/window/hostInput.cpp
3	16	src/graphics/presentation/window/swapchain.cpp
119	165	src/graphics/presentation/window/vulkanWindow.cpp
22	17	src/graphics/presentation/window/window.cpp
0	77	src/graphics/rt/hardware.cpp
0	10	src/graphics/rt/hardware.h
1	16	src/graphics/shader/recompiler/BufferFormat.h
84	154	src/graphics/shader/recompiler/ShaderRecompiler.cpp
0	1	src/graphics/shader/recompiler/ShaderRecompiler.h
342	0	src/graphics/shader/recompiler/Tessellation.cpp
25	0	src/graphics/shader/recompiler/Tessellation.h
45	109	src/graphics/shader/recompiler/backend/spirv/SpirvBuilder.cpp
106	31	src/graphics/shader/recompiler/backend/spirv/SpirvBuilder.h
20	32	src/graphics/shader/recompiler/backend/spirv/SpirvEmitter.cpp
0	2	src/graphics/shader/recompiler/backend/spirv/SpirvEmitter.h
389	575	src/graphics/shader/recompiler/backend/spirv/spirvEmitterAlu.cpp
80	207	src/graphics/shader/recompiler/backend/spirv/spirvEmitterAluHelpers.cpp
63	64	src/graphics/shader/recompiler/backend/spirv/spirvEmitterAnalysis.cpp
380	330	src/graphics/shader/recompiler/backend/spirv/spirvEmitterFlow.cpp
92	84	src/graphics/shader/recompiler/backend/spirv/spirvEmitterHelpers.cpp
246	237	src/graphics/shader/recompiler/backend/spirv/spirvEmitterImage.cpp
304	0	src/graphics/shader/recompiler/backend/spirv/spirvEmitterInstructions.h
157	441	src/graphics/shader/recompiler/backend/spirv/spirvEmitterInternal.h
527	583	src/graphics/shader/recompiler/backend/spirv/spirvEmitterMemory.cpp
100	139	src/graphics/shader/recompiler/backend/spirv/spirvEmitterMemoryHelpers.cpp
89	80	src/graphics/shader/recompiler/backend/spirv/spirvEmitterMesh.cpp
376	575	src/graphics/shader/recompiler/backend/spirv/spirvEmitterModule.cpp
254	181	src/graphics/shader/recompiler/backend/spirv/spirvEmitterProgram.cpp
555	0	src/graphics/shader/recompiler/backend/spirv/spirvEmitterRaytracing.cpp
161	0	src/graphics/shader/recompiler/backend/spirv/spirvEmitterTessellation.cpp
107	102	src/graphics/shader/recompiler/frontend/cfg/ShaderCFG.cpp
10	0	src/graphics/shader/recompiler/frontend/cfg/ShaderCFG.h
0	1	src/graphics/shader/recompiler/frontend/decode/ExportOps.cpp
142	158	src/graphics/shader/recompiler/frontend/decode/ImageOps.cpp
8	17	src/graphics/shader/recompiler/frontend/decode/MemoryOps.cpp
25	8	src/graphics/shader/recompiler/frontend/decode/ScalarAluOps.cpp
75	29	src/graphics/shader/recompiler/frontend/decode/ShaderDecoder.cpp
31	5	src/graphics/shader/recompiler/frontend/decode/ShaderDecoder.h
41	103	src/graphics/shader/recompiler/frontend/decode/VectorAluOps.cpp
3	1	src/graphics/shader/recompiler/frontend/translate/Compare.cpp
78	29	src/graphics/shader/recompiler/frontend/translate/Control.cpp
4	4	src/graphics/shader/recompiler/frontend/translate/Convert.cpp
11	14	src/graphics/shader/recompiler/frontend/translate/Float.cpp
34	8	src/graphics/shader/recompiler/frontend/translate/Integer.cpp
96	34	src/graphics/shader/recompiler/frontend/translate/Memory.cpp
32	1	src/graphics/shader/recompiler/frontend/translate/Scalar.cpp
166	123	src/graphics/shader/recompiler/frontend/translate/Translate.cpp
1	9	src/graphics/shader/recompiler/frontend/translate/Translate.h
14	10	src/graphics/shader/recompiler/frontend/translate/Translator.h
11	0	src/graphics/shader/recompiler/frontend/translate/Vector.cpp
29	17	src/graphics/shader/recompiler/ir/Program.cpp
25	23	src/graphics/shader/recompiler/ir/ShaderIR.h
3	0	src/graphics/shader/recompiler/ir/opcodes/ValueOpcodes.cpp
4	3	src/graphics/shader/recompiler/ir/opcodes/ValueOpcodes.h
12	0	src/graphics/shader/recompiler/ir/opcodes/ValueOpcodes.inc
9	0	src/graphics/shader/recompiler/ir/passes/ConstantPropagation.cpp
51	16	src/graphics/shader/recompiler/ir/passes/ResourceMaterialization.cpp
6	0	src/graphics/shader/recompiler/ir/passes/ResourceMaterialization.h
103	22	src/graphics/shader/recompiler/ir/passes/ResourceTracking.cpp
73	32	src/graphics/shader/recompiler/ir/passes/ShaderInfoCollection.cpp
1	7	src/graphics/shader/recompiler/ir/passes/ShaderInfoCollection.h
399	34	src/graphics/shader/recompiler/ir/passes/SrtWalker.cpp
6	0	src/graphics/shader/recompiler/ir/passes/SrtWalker.h
2	42	src/graphics/shader/recompiler/ir/passes/SsaRewrite.cpp
104	124	src/graphics/shader/rectListShader.cpp
116	38	src/graphics/shader/shader.cpp
37	6	src/graphics/shader/shader.h
4	0	src/graphics/shader/shaderBindings.h
4	0	src/graphics/shader/shaderCompiler.h
19	7	src/graphics/shader/shaderPixelParameter.cpp
44	7	src/kernel/eventFlag.cpp
128	197	src/kernel/fileSystem.cpp
107	17	src/kernel/memory.cpp
3	2	src/kernel/memory.h
19	28	src/kernel/memoryAddressSpace.inc
113	307	src/kernel/pthread.cpp
0	1	src/kernel/pthread.h
20	11	src/kernel/syncOnAddress.cpp
4	0	src/kernel/syncOnAddress.h
2	2	src/kytylibrary/CMakeLists.txt
41	27	src/kytylibrary/forms/configuration_edit_dialog.ui
5	0	src/kytylibrary/forms/configuration_list_widget.ui
0	2	src/kytylibrary/include/compatibilityDatabase.h
20	14	src/kytylibrary/include/configuration.h
0	6	src/kytylibrary/include/configurationEditDialog.h
0	4	src/kytylibrary/include/configurationItem.h
1	2	src/kytylibrary/include/mandatoryLineEdit.h
0	4	src/kytylibrary/include/patchesDialog.h
0	4	src/kytylibrary/include/trophyViewerDialog.h
0	2	src/kytylibrary/qml/Library.qml
15	1	src/kytylibrary/qml/SettingsPage.qml
79	20	src/kytylibrary/src/configurationEditDialog.cpp
39	7	src/kytylibrary/src/configurationItem.cpp
7	5	src/kytylibrary/src/configurationListWidget.cpp
12	1	src/kytylibrary/src/emulatorArguments.cpp
2	2	src/kytylibrary/src/inputMappingDialog.cpp
63	14	src/kytylibrary/src/librarySettings.cpp
7	3	src/kytylibrary/src/patchesDialog.cpp
12	25	src/kytylibrary/src/trophyViewerDialog.cpp
6	2	src/kytylibrary/translations/en.json
6	2	src/kytylibrary/translations/it.json
12	21	src/libs/agc.cpp
13	2	src/libs/ajm.cpp
32	22	src/libs/ajm/aac_decoder.h
1	0	src/libs/ajm/decoder.h
211	62	src/libs/audio.cpp
4	1	src/libs/audio.h
37	7	src/libs/avPlayer.cpp
112	2	src/libs/controller.cpp
3	0	src/libs/controller.h
6	1	src/libs/guestPrintf.cpp
24	0	src/libs/hmd2.h
26	29	src/libs/ime.cpp
22	25	src/libs/imeDialog.cpp
46	179	src/libs/libAmpr.cpp
5	5	src/libs/libAppContent.cpp
3	1	src/libs/libAudio.cpp
323	0	src/libs/libCes.cpp
1	0	src/libs/libDbgAsan.cpp
13	2	src/libs/libFont.cpp
240	0	src/libs/libHmd2.cpp
86	1	src/libs/libJson2.cpp
124	168	src/libs/libKernel.cpp
50	225	src/libs/libNet.cpp
5	9	src/libs/libPsml.cpp
7	7	src/libs/libSaveData.cpp
3	1	src/libs/libSystemService.cpp
7	0	src/libs/libs.cpp
324	99	src/libs/network.cpp
3	0	src/libs/network.h
7	16	src/libs/videoDec2Decoder.cpp
1	0	src/loader/gamePatch.cpp
531	428	src/loader/redZonePatcher.cpp
7	3	src/loader/redZonePatcher.h
90	37	src/loader/runtimeLinker.cpp
7	7	src/loader/symbolDatabase.cpp
7	8	src/loader/systemContent.cpp
0	8	src/loader/timer.cpp
0	2	src/loader/timer.h
239	269	src/loader/x64InstructionEmulator.cpp
6	0	src/loader/x64InstructionEmulator.h
25	12	src/main.cpp
0	21	src/utils.cmake
7	9	tests/ImeDialogTests.cpp
237	3	tests/KernelFileSystemTests.cpp
35	22	tests/MemoryTrackerTests.cpp
237	7	tests/ResourceTrackingTests.cpp
5641	628	tests/ShaderRecompilerComputeTests.cpp
33	0	tests/SyncOnAddressTests.cpp
537	22	tests/VirtualMemoryAllocationTests.cpp
20	0	tests/library_ui_tests.cpp
219	0	tests/macosSse4aTests.cpp
2100	1305	tests/shaderCfgTests.cpp
1	1	version.cmake
```

## Appendice B — integrità dei log analizzati

- `Build-Fran/gta5-log.txt`: 10731824 byte, SHA-256 `a9c9c3a093ba7e6cc3b3541bc6968e0dfeaf8e227d635503d80d5b9310d261fb`.
- `Build-Fran1/gta5-log.txt`: 138457280 byte, SHA-256 `4b6fdae59a9cc939d073b42f8e8eb0d436ec1119ac42917436d15916068653d0`.
