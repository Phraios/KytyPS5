# Continuazione post-revisione — 10 settembre 2026 (2)

**LEGGERE PRIMA QUESTA SEZIONE.** Stato lasciato per la ripresa: la parte
barrier è finita e verificata nei test; NON è stata reinstallata né provata
nel gioco. Nessun commit né push. Dettagli sotto; la revisione di Astra resta
valida dopo questa sezione.

## Cosa è stato fatto in questa continuazione

1. Test dispatcher numerici (chiesto al punto 2 della revisione): aggiunti
   `DispatcherPhiUntakenArm`, `DispatcherPhiTakenArm`, `DispatcherPhiSwapVgpr`
   in tests/ShaderRecompilerComputeTests.cpp, varianti Wave32+Wave64, più
   selettore `--dispatcher-phi-regressions`. Verificata empiricamente la
   sensitività togliendo il fix: `PhiSwap` e `SwapVgpr` (entrambe le wave)
   falliscono senza fix; `Taken/UntakenArm` passano in entrambi i casi e
   restano come guardie direzionali (else-edge / taken-edge), con commenti
   onesti nel codice. Durante i test trovati e corretti DUE bug nei test
   stessi, non nel ricompilatore: literal oltre 64 in `InlineU32` e costanti
   inline in `src1` di VOP2 (lì sono ammessi solo VGPR; il decoder fa bene).
2. Fix barriera VS (nuovo blocco trovato con la prova controllata): un vertex
   shader di Astro Bot (hash 0xd4bac36b6b1f31c5, 0 loop) falliva la validazione
   con OpControlBarrier a scope Workgroup, illegale in VS. Mai compilato nei
   run precedenti (0 riferimenti), quindi preesistente ed esposto ora.
   `EmitValueFlow` in spirvEmitterFlow.cpp usa scope Subgroup per gli stage
   non-compute (Compute e Mesh invariati). Regressione
   `TestNewShaderRecompilerVertexBarrierSubgroupScope` in shaderCfgTests.cpp
   (OpControlBarrier presente, scope Subgroup, SPIR-V valida).
3. Suite completa: 40/40 CTest verdi con tutto dentro.
4. Correlazione dispatch (punto 4, versione minima e bounded): la riga
   GraphicsRenderDispatchDirect ora riporta `id=` monotonico, `hash=` dello
   shader e frame/indirizzo guest; invariati filtro e tetto a 512 righe,
   nessuna modifica di semantica.
5. Prova controllata di Astro Bot con validazione: TileBasedLighting via
   dispatcher valida ed eseguita (36 dispatch osservati in precedenza), poi il
   run è morto sul VS barrier di cui sopra (blocco ora corretto nei test, MAI
   ancora provato nel gioco).

## Pulizia eseguita su richiesta

Rimossi tutti i file temporanei di sessione (Temp/opencode svuotata: eseguibili
copiati, log di gioco/test, script helper) e l'estratto in Build-Fran già
rimosso prima. Nel codice non restano dump env-gated, marker né debug
(verificato via grep: solo match preesistenti). Resta solo il lavoro vero,
elenco sotto.

## Per riprendere (stato lasciato)

Worktree con modifiche NON committate né caricate (git status):
- src/graphics/shader/recompiler/backend/spirv/spirvEmitterProgram.cpp
  (fix Phi simultanee + rami condizionali/indiretti, sessione revisione)
- src/graphics/shader/recompiler/backend/spirv/spirvEmitterFlow.cpp
  (scope Subgroup barriera VS/PS, questa continuazione)
- src/graphics/shader/recompiler/ShaderRecompiler.cpp
  (fallback bare-branch, sessione precedente)
- src/graphics/shader/recompiler/frontend/cfg/ShaderCFG.cpp/.h
  (FindUnemittableBareBranch, sessione precedente)
- src/graphics/host_gpu/renderer/renderCompute.cpp (id/hash dispatch)
- src/graphics/host_gpu/renderer/cache/bufferCache.cpp (fence diagnostica)
- src/graphics/host_gpu/renderer/pipeline/pipelineCache.cpp,
  ResourceMaterialization.cpp/.h, SrtWalker.cpp/.h (diagnostica + SRT)
- tests/ShaderRecompilerComputeTests.cpp (7 regressioni dispatcher)
- tests/shaderCfgTests.cpp (LoopExitTailDispatcher + VertexBarrier)
- tests/ResourceTrackingTests.cpp, docs/astrobot-handoff.md

Aperti, in ordine:
1. Reinstallare Build-Fran (l'installazione attuale NON contiene fix barrier
   né riga dispatch id/hash: rebuild + `cmake --install`) e rifare la prova
   controllata di Astro Bot con validazione: atteso superamento del VS
   0xd4bac36b6b1f31c5, poi si vedrà se il fence/TDR resta.
2. Test numerico per rami indiretti saltato di proposito: il selettore
   jump-table richiede s_load dal codice con memorie di test incoerenti
   (shader_base vs initial); i percorsi indiretti condividono l'helper già
   coperto dai test condizionali taken/untaken.
3. Poi i punti 5-6 della revisione (fence esatta, BDA). TdrDelay invariato,
   validazione sempre attiva. Non rilanciare il gioco alla cieca.

# Revisione crash GPU — 10 settembre 2026

**LEGGERE DOPO LA SEZIONE SOPRA.** Aggiornamento precedente al resoconto sotto.
L'utente ha chiesto di fermare l'implementazione e lasciare al prossimo agente
l'analisi, il difetto trovato e i passi per continuare.

## Conclusione attuale

È stato riprodotto un BUG REALE nel dispatcher SPIR-V preesistente, ora usato
anche dagli shader Astro Bot grazie al fallback aggiunto dal collega.
Non è ancora dimostrato che questo bug sia la causa del reset GPU del gioco.
Non dichiarare il crash risolto. Nessuna nuova prova Astro Bot è stata eseguita
in questa sessione di revisione e nessun limite TDR è stato modificato.

## Valutazione del codice nel worktree

Esaminati i cambiamenti a ShaderRecompiler.cpp, ShaderCFG.cpp/.h,
pipelineCache.cpp, ResourceMaterialization.cpp/.h, SrtWalker.cpp/.h,
ResourceTrackingTests.cpp e shaderCfgTests.cpp, più i percorsi richiamati
nel dispatcher SPIR-V e nella buffer cache.

- Il fallback del collega è una soluzione ragionevole al precedente SPIR-V
  non valido: riparte dal CFG originale, usa il dispatcher esistente e conserva
  la validazione. Non è un bypass ray tracing.
- La regressione aggiunta dal collega verifica selezione del dispatcher e
  validità SPIR-V; NON ne verifica il risultato numerico su GPU. La validità
  strutturale non garantisce la semantica corretta dello shader.
- Il controllo FindUnemittableBareBranch è mirato ai casi osservati, non una
  dimostrazione generale che ogni ramo rimanente sia emettibile. La validazione
  SPIR-V deve restare attiva. Le affermazioni assolute nel resoconto storico
  su tutti gli shader validi/non validi sono più forti dell'evidenza disponibile.
- Diagnostica MaterializeResources/SrtWalker: utile, errori conservati per thread
  e log locali; nessuna nuova causa diretta di reset GPU individuata lì.
- La correzione SRT della sessione precedente mantiene sulla GPU puntatori e
  letture dinamiche prima anticipati sulla CPU. È coerente con il problema
  osservato, ma aumenta l'importanza di verificare BDA e controllo di flusso
  durante l'esecuzione reale. I test SRT non provano la sicurezza di ogni accesso
  GPU di Astro Bot.
- Nel complesso: avanzamento utile e motivato; la verifica del percorso
  dispatcher sotto carico 3D è incompleta. Il bug sotto non è stato introdotto
  dal collega, ma il suo fallback lo rende raggiungibile in più shader.

## BUG DIMOSTRATO: assegnazioni Phi non simultanee

File: src/graphics/shader/recompiler/backend/spirv/spirvEmitterProgram.cpp
Funzione: StoreDispatcherPhiEdge.

Prima della correzione, per ogni Phi il codice emetteva immediatamente:

    OpStore destination_spill, ctx.Def(incoming_value)

Passava poi alla Phi successiva. Se i valori delle Phi devono essere scambiati,
la prima scrittura può sovrascrivere il valore che la seconda deve ancora leggere.
Le Phi devono comportarsi come assegnazioni simultanee, non sequenziali.

Riproduzione minima aggiunta in tests/ShaderRecompilerComputeTests.cpp:
DispatcherPhiSwap, loop limitato a TRE iterazioni, inizialmente a=1 e b=2,
con scambio a/b ad ogni iterazione. Il contatore è indipendente: il test non è
un loop infinito e non è stato necessario provocare un TDR per riprodurlo.
Il test forza il dispatcher solo nel test harness.

PRIMA, risultato GPU realmente osservato:

    expected [0x00000002, 0x00000001]
    actual   [0x00000001, 0x00000001]

Log: _Build/gpu-review-phi.log

DOPO la correzione preparata:

    [compute] DispatcherPhiSwap ok

Log: _Build/gpu-review-phi-fixed.log

Questo dimostra corruzione del risultato del dispatcher. Se gli stessi valori
sono puntatori o stato di controllo di un loop, può produrre letture errate o
loop che non terminano: è un collegamento PLAUSIBILE al crash, non confermato
sul gioco specifico.

## Correzioni già preparate, NON ancora validate completamente

Quando l'utente ha chiesto di fermarmi, avevo già scritto nel worktree:

1. spirvEmitterProgram.cpp:
   - StoreDispatcherPhiEdge raccoglie tutte le coppie destinazione/valore,
     valutando prima tutti gli operandi, poi emette le scritture.
   - Gli aggiornamenti Phi dei rami condizionali/indiretti avvengono soltanto
     sul ramo selezionato, tramite selezioni SPIR-V strutturate. Prima venivano
     aggiornati i target di entrambi i rami prima di scegliere il prossimo PC.
   - La condizione/il selettore viene calcolato prima degli aggiornamenti.
   La riproduzione sopra prova il difetto di assegnazione simultanea; la modifica
   agli archi condizionali/indiretti necessita ancora di regressioni dedicate.

2. ShaderRecompilerComputeTests.cpp:
   - campo force_dispatcher nel SOLO TestCase;
   - DispatcherPhiSwap registrato nella suite compute;
   - selettore --dispatcher-phi-only per eseguire la regressione isolata.

3. bufferCache.cpp:
   - l'errore della fence stampa il risultato Vulkan esatto (nome e numero),
     tick, numero copie e byte. Il comportamento resta fail-fast sull'errore.

Build tramite _Build/build-upstream.cmd riuscita.
Solo il test --dispatcher-phi-only è stato eseguito dopo queste modifiche.
NON è stata eseguita la suite completa, NON è stato rilanciato Astro Bot,
NON è stata installata questa build in Build-Fran, NON sono stati fatti commit
né push. Le modifiche precedenti del collega sono preservate.
Log build: _Build/gpu-review-build.log.

## Errori nell'interpretazione precedente del crash

### Il frame 39 non identifica il punto del blocco

File: src/graphics/host_gpu/renderer/renderCompute.cpp, GraphicsRenderDispatchDirect.
Il log dettagliato dei dispatch ha un contatore statico con limite < 512 e
registra solo alcuni dispatch (large_workgroup || has_sampler).
Quindi «512 dispatch, poi nessun dispatch dopo il frame 39» significa che il
contatore ha smesso di stampare. NON prova che l'esecuzione GPU si sia fermata
al frame 39. Anche «36 dispatch TileBasedLighting eseguiti» va inteso come
messaggi host osservati, non prova di completamento GPU di 36 dispatch.

Il log attuale ../Build-Fran/astrobot-log.txt contiene circa 778 mila righe,
termina alle 11:01:05 del 10 settembre, e registra la stessa failure fence.
Sono presenti eventi nvlddmkm 153 alle 11:01:05 e alle 10:38:24.
La ripetibilità aiuta a isolare il trigger, ma non distingue da sola un errore
ripetuto da un singolo dispatch che si blocca in modo riproducibile.

### La failure fence non prova un normale timeout

La vecchia riga è:

    device.waitForFences(1, &m_readback_fence, VK_TRUE, UINT64_MAX) != eSuccess

Il timeout richiesto è UINT64_MAX. Il codice scarta il valore di ritorno:
può essere errore del dispositivo, non necessariamente eTimeout.
Gli eventi del driver e questa failure sono coerenti con perdita/reset del
contesto GPU, ma dal vecchio log non si ricava il risultato Vulkan effettivo.
La modifica diagnostica preparata sopra serve esattamente a conservarlo.
La fence è il punto in cui la CPU osserva il fallimento; non identifica lo shader
che lo ha causato.

### Hardware non escluso dall'analisi

Le affermazioni storiche sotto («escluso hardware», «indagine conclusa»,
«VRAM bacata farebbe crashare anche i giochi normali», «nessun danno possibile»)
NON sono conclusioni dimostrate da questi log. Non attribuire il crash al driver,
al dispatcher o all'hardware con certezza prima delle verifiche.

## Come continuare, in ordine

1. Rivedere il diff di spirvEmitterProgram.cpp: verificare snapshot simultaneo
   degli operandi Phi, entrambe le metà wave64, cache dei load di spill e label
   dopo le nuove selezioni. Conservare la regressione che falliva prima.
2. Aggiungere test NUMERICI GPU limitati per archi condizionali non presi,
   archi indiretti e Phi cicliche, wave32/wave64. Non usare loop infiniti come
   test e non considerare la sola validazione SPIR-V sufficiente.
3. Eseguire la suite completa: _Build/test-upstream.cmd (40 CTest, più il nuovo
   caso all'interno della suite compute). Risolvere eventuali regressioni.
4. Prima di una nuova prova del gioco, migliorare la correlazione diagnostica:
   ID dispatch, hash shader (program.shader_hash), indirizzo guest, frame e tick
   scheduler. Il log attuale mostra indirizzo guest e si tronca dopo 512 eventi;
   non basta per individuare l'ultimo lavoro GPU completato.
   Preferire un tracciamento circoscritto/ring buffer e checkpoint/completamenti,
   senza cambiare la semantica o saltare shader.
5. Solo dopo i test, installare e fare una prova controllata di Astro Bot con
   validazione attiva, registrando l'esito esatto della fence. Non rilanciare
   ripetutamente il gioco alla cieca provocando reset del driver.
6. Se il crash resta, proseguire su accessi BDA, sincronizzazioni e lavoro
   effettivamente in volo. Non aumentare TdrDelay e non disattivare la validazione
   per mascherare il problema.

---

# Resoconto storico della sessione precedente

Le sezioni seguenti sono conservate come contesto. Dove contraddicono la
revisione qui sopra, prevalgono le evidenze e i limiti della revisione nuova.

# Astro Bot — handoff

Aggiornato: 10 settembre 2026, dopo la correzione del controllo di flusso SPIR-V.

## Stato attuale

Il blocco SRT è risolto (vedi sotto): Astro Bot supera MaterializeResources
nello shader compute TileBasedLighting, hash 0x78af8e269b528b5c.

Il blocco successivo — validazione SPIR-V "Selection must be structured"
(`OpBranchConditional %28671 %180 %117`) — è RISOLTO con il fallback
dispatcher: lo shader compila, valida, crea la pipeline e viene eseguito
(36 dispatch osservati). Il gioco progredisce fino al frame 39 con 512
dispatch compute e continua lo streaming degli asset.

Nuovo blocco, SEPARATO: dopo ~1,9M righe di log il gioco muore su timeout
della fence in `bufferCache.cpp` durante lo streaming, preceduto da fault GPU
(Evento nvlddmkm 153). Indagine dedicata sotto: punta a driver/app, non
all'hardware. Il gioco NON arriva ancora alla scena 3D. La validazione resta
attiva e il supporto ray tracing di Brandon rimane operativo, senza bypass.

## Repository e modifiche

HEAD main: 9678765, merge upstream fino a 0b4e78c.
Il supporto Brandon astrobot/rt (tip 25da84b) è già nel merge 589ae76.
Il precedente merge upstream ha risolto il test depth/stencil: non è più un blocco.

Le modifiche di questa sessione NON sono state committate né caricate:
- Diagnostica preesistente dell'altra IA conservata in pipelineCache.cpp,
  ResourceMaterialization.cpp/.h e SrtWalker.cpp/.h.
- Correzione del planner SRT in SrtWalker.cpp (sessione precedente, utente).
- Due regressioni aggiunte a tests/ResourceTrackingTests.cpp (sessione
  precedente, utente).
- Correzione SPIR-V di questa sessione: `FindUnemittableBareBranch` in
  ShaderCFG.h/.cpp + instradamento al dispatcher in ShaderRecompiler.cpp.
- Una regressione aggiunta a tests/shaderCfgTests.cpp:
  `TestNewShaderRecompilerCfgLoopExitTailDispatcher`.
- Questo handoff aggiornato.

Non annullare globalmente il worktree: contiene la diagnostica dell'altra IA
insieme alle correzioni e ai test. Nessuna modifica finale a
spirvEmitterProgram.cpp (i dump temporanei usati per l'analisi sono stati
rimossi tutti).

## Causa e correzione del blocco SRT

(Riassunto invariato dalla sessione precedente.)
Il log originale riconfermava source 22/dword 0, buffer scalare usato a
PC 0x2150. L'indirizzo dipendeva da Phi e da letture a PC 0x2030: non era un
descrittore calcolabile anticipatamente sulla CPU. Il planner scambiava un
offset costante per una lettura completamente costante; dopo la prima
correzione emergevano letture anticipate non valide a PC 0x1f90 e 0x1f6c.
Correzione finale: verifica ValidateRuntimeValue prima di appiattire una
lettura; esclusione dei GetAddressResource dalla raccolta diretta (consumati
dalla GPU); buffer descriptor dinamici sul percorso GPU; ottimizzazione delle
letture dati limitata ai blocchi iniziali incondizionati; nessun descrittore
nullo per nascondere errori. Regressioni: TestDynamicPointerChaseUsesDma,
TestGuardedScalarDataRemainsOnGpu.

## Causa e correzione del blocco SPIR-V

Shader 0x78af8e269b528b5c: il CFG risulta strutturato (198 blocchi, IR 199),
ma il blocco 51 (label SPIR-V %116, primo blocco del body) termina con
`OpBranchConditional` senza `OpSelectionMerge` verso la tail 115 (label %180),
che raggiunge il merge 153 (label %218) del loop 50. Il validatore risponde
"Selection must be structured".

Analisi: un ramo nudo è valido solo verso merge/continue/header del loop
(break/continue diretti, confermati dai test esistenti). Un ramo verso una
tail non lo è mai. `IsInnermostLoopControlConditional` classificava invece
come loop-control qualunque uscita con un solo braccio nel body, incluso
`true_in_body != false_in_body` verso tail arbitrarie.

Tentativi scartati (NON nel codice finale): restringere
`IsInnermostLoopControlConditional` rompeva
`TestNewShaderRecompilerCfgNestedLoopExitTailMergeSplit` in tutti i modi
provati (rimozione del caso, restrizione ai target diretti, regole di
reachability/backedge con varie forme di stop al merge); condividere il merge
del loop come selection merge è illegale in SPIR-V ("already a merge block
for another header", verificato sperimentalmente). La strutturazione di
un'uscita-via-tail che si unisce al merge del proprio loop non è
rappresentabile senza duplicare il loop.

Correzione finale (minima, nessun cambio al classificatore):
1. `CFG::FindUnemittableBareBranch` (ShaderCFG.h/.cpp): dopo Structurize con
   successo, segnala il primo condizionale senza merge i cui bracci non
   restano nel body del loop più interno né puntano al suo merge.
2. `ShaderRecompiler.cpp`: in quel caso scatta lo stesso percorso del
   fallback dispatcher esistente (fase loggata come `bare-branch`), con
   validazione SPIR-V comunque attiva. Nessun bypass, nessun rilassamento.
3. Regressione `TestNewShaderRecompilerCfgLoopExitTailDispatcher`: uscita via
   tail su fixture minima → dispatcher (OpSwitch) + SPIR-V valido. Il test
   nested resta intatto (chiama solo Structurize, mai toccato).

Questa scelta non cambia nessuno shader oggi valido: per costruzione, un ramo
nudo verso una tail non valida mai, quindi il pre-check non dirotta shader
funzionanti (suite 40/40 a riprova, inclusi tutti i conteggi OpSelectionMerge
esistenti).

## Verifiche e installazione

Build Windows riuscita tramite build-kyty.bat + target kyty_emulator.
Suite CTest completa: 40/40 passati.
Installazione in W:/KytyLab/KytyPS5/Fran/Build-Fran (binari verificati con
timestamp e stringhe del fallback).

Prova Astro Bot con shader validation attiva (log completo rimosso durante
la pulizia finale; evidenze chiave ricopiate qui):
- `CFG dispatcher fallback ... phase=bare-branch` per 0x78af8e269b528b5c e
  altri 3 shader mai raggiunti prima (avrebbero fallito la validazione allo
  stesso modo).
- SPIR-V emesso (248945 word per TileBasedLighting), validazione PASSATA,
  `vkCreateComputePipelines Success`, primo dispatch al frame 3
  (`groups=1x1x1`, 8 buffer, 17 texture).
- 512 dispatch compute fino al frame 39, TileBasedLighting eseguito 36 volte,
  pipeline create con successo fino a poche centinaia di righe dal crash.

Log precedenti preservati: astrobot-log.txt (originale),
astrobot-srt-verified-log.txt (+ stderr). I log pointer-fix*, srt-final e
simili restano prove intermedie. I log temporanei della sessione dispatcher
sono stati rimossi su richiesta.

## Nuovo blocco separato: timeout fence / TDR in streaming

Dopo il frame 39 i dispatch si fermano (~riga 192639) e il gioco continua con
solo streaming asset per ~73k righe, poi muore due volte su:
`device.waitForFences(1, &m_readback_fence, ...) != eSuccess` in
`graphics/host_gpu/renderer/cache/bufferCache.cpp`. Il driver ha registrato 7
fault GPU (Evento nvlddmkm 153, "Error occurred on GPUID: 100") alle 10:38:24
del 10/09: TDR con reset driver, scheda illesa e operativa dopo (35 °C idle).

Indagine causa scatenante (chiesta dall'utente, conclusa):
- Episodio precedente il 02/09: Minecraft moddato crashato DENTRO nvoglv64.dll
  (driver OpenGL, 0xc0000409) + 19 Eventi 153 a raffica. Stessa firma: app che
  sottomette ogni frame lavoro anomalo.
- Escluso overclock (Afterburner a stock, boost 0, power limit 80%),
  escluso calore (mai sopra 60 °C), zero errori WHEA, zero TDR di sistema
  (4101/141), Kernel-Power 41 tutti legati allo sleep (altro problema).
- Diagnosi: in entrambi i casi lavoro GPU anomalo (mod/shaderpack;
  shader sperimentali dell'emulatore, incluso il dispatcher OpSwitch) che
  manda in fault il driver. Ipotesi hardware (VRAM) in fondo alla lista:
  con VRAM bacata crasherebbero anche i giochi normali. Nulla di permanente:
  da userspace non si danneggia la scheda.
- Resta aperto SE il fault venga da uno dei 4 shader dispatcher (bug di
  emissione non rilevato dalla validazione) o da altro codice preesistente:
  dal log non si distingue (manca il legame indirizzo→hash nei dispatch).
  Prossimo passo possibile, solo su richiesta: run con validation layer GPU o
  bisezione degli shader (pesante per la GPU).

## Ambiente, sicurezza e recupero

Repository: W:/KytyLab/KytyPS5/Fran/KytyPS5-Fran
Gioco: W:/KytyLab/Games/AstroBot
Build: _Build/windows
Helper locali: build-kyty.bat (libreria+install), _Build/build-upstream.cmd
(test target), _Build/test-upstream.cmd (CTest; configura Qt offscreen
6.10.3). Per l'emulatore aggiornato serve anche
`cmake --build _Build/windows --target kyty_emulator` prima di
`cmake --install _Build/windows --prefix <Build-Fran>`: build-kyty.bat da
solo compila kyty_library.

Defender aveva segnalato Bearfoos.A!ml durante la precedente sessione
diagnostica. Nessuna esclusione aggiunta in questa sessione; i binari usati
per i test sono quelli installati da build-kyty.bat in Build-Fran.

Backup precedenti: _Build/pre-brandon-rt-20260909/ e
_Build/pre-upstream-20260909/.
Lo stash "Preserve Fran local UI edits and handoff before Brandon RT merge" era
già stato applicato: non riapplicarlo automaticamente.
