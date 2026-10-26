---
marp: true
theme: presentazione
paginate: true
---

<style scoped>
section { padding: 50px 60px; font-size: 20px; justify-content: center; }
.label { font-size: 0.7em; letter-spacing: 0.18em; text-transform: uppercase; color: var(--blue); font-weight: bold; margin: 0 0 14px 0; }
h1 { text-align: left; font-size: 1.45em; line-height: 1.25; margin: 0 0 18px 0; padding-bottom: 18px; border-bottom: 6px solid var(--blue); }
.people { display: grid; grid-template-columns: auto 1fr; gap: 6px 18px; margin: 0 0 28px 0; }
.people .role { font-size: 0.7em; letter-spacing: 0.08em; text-transform: uppercase; color: var(--blue); align-self: center; font-weight: bold; }
.people .name { font-size: 1.05em; }
.uni { font-size: 0.8em; color: #555; line-height: 1.5; margin: 0; }
.uni b { color: #222; }
</style>


<p class="label">Tesi di Laurea Magistrale</p>

# Implementazione di una pipeline per la generazione di mesh 3D a partire da modelli CAD 2D di planimetrie architettoniche

<div class="people">
<span class="role">Candidato</span><span class="name"><b>Simone Vaiasinni</b></span>
<span class="role">Relatore</span><span class="name">Prof. Nicola Capodieci</span>
<span class="role">Correlatore</span><span class="name">Prof. Fabio Pellacini</span>
</div>

<p class="uni"><b>Laurea Magistrale in Informatica</b><br>
Università degli Studi di Modena e Reggio Emilia · A.A. 2025/2026</p>

---

<!-- 
DESCRIZIONE DEI PROBLEMI

2. le finestre a volte venivano rappresentate come buchi tra i muri mentre altre volte venivano sovrapposte ai muri, a volte veniva rappresentato il vetro interno e altre volte no, ecc.

3. le porte a volte rappresentate con l’anta e l’arco di apertura mentre altre come porta battente a 2 ante, altre volte possono essere rappresentati come dei rettangoli.

4. la geometria presentasse talvolta del rumore e imperfezioni. Succedeva che erano presenti segmenti/punti residui lasciati durante la modifica del disegno, oppure i muri risultavano "rotti" (aperti) e non erano dei poligoni chiusi, estremità non coincidenti per piccole distanze

5. succedeva che nel caso delle porte queste venissero rappresentate o salvate in modo diverso, perché quando provavo a fare il parsing su modello (draftperson) funzionava mentre se provavo su un altro modello (archweb) non funzionava più nel senso che i segmenti venivano plottati in modo errato nonostante la procedura fosse uguale per entrambi.

-->

<style scoped>
section { font-size: 20px; }
h1 { margin-bottom: 22px; line-height: 1.5; }
.row { grid-template-columns: 1fr 1.2fr; align-items: stretch; gap: 12px 18px; }
.col { gap: 12px; }
.col .card { flex: 1; }
.row > .box { height: 100%; }
</style>

# Introduzione: dal CAD 2D al modello 3D

<div class="row">
<div class="col">

<div class="box" style="flex:1">

## Obiettivo

Da una planimetria CAD di una stanza o di un appartamento generare un **modello 3D** e renderizzarlo in **Blender** con materiali e asset.

</div>

<div class="box green" style="flex:1">

## La sfida

L'algoritmo deve:
- capire **quali segmenti formano un muro**, dove c'è un'**apertura** e ricostruirla
- capire quali **blocchi** sono porte o finestre e dove posizionarli

</div>

</div>

<div class="box red">

## Difficoltà

- Un CAD contiene solo **geometria** (linee, archi, polilinee, blocchi), non la **semantica** dell'edificio
- Un autore può utilizzare una convenzione diversa e rappresentare gli elementi in modi differenti.
- **Layer** con nomi diversi, anche in lingue diverse
- **Rumore e imperfezioni** da rimuovere o correggere
- **Unità di misura** diverse
- Alterazioni nella **conversione DWG → DXF**

</div>
</div>

---

<style scoped>
.row { grid-template-columns: 1.4fr 1fr; gap: 24px; align-items: center; }
section { --img-h: 400px; }
</style>

# Passo 0: Preprocessing

<p class="goal">L'obiettivo è pulire il disegno e renderlo più <b>consistente</b></p>

<div class="row">
<figure>
<img src="figures/3_progettazione_pipeline/caso-studio.png">
<figcaption>Modello di partenza: disegno, layer e blocchi</figcaption>
</figure>

<div class="col">

<div class="card">

## 1. Nomi

Layer e blocchi rinominati: `A-WALL` `A-DOOR` `A-WINDOW` `BLOCK-DOOR` `BLOCK-WINDOW`

</div>

<div class="card orange">

## 2. Pulizia

Rimossi gli elementi inutili o problematici (es. `WALL-EXTERNAL`)

</div>

<div class="card green">

## 3. Correzione

Contorni dei muri chiusi e blocchi delle finestre ripuliti

</div>

</div>
</div>

---

<style scoped>
section { --img-h: 360px; --cap: 1em; }
.row { align-items: start; }
</style>


<div class="row">
<figure>
<img src="figures/3_progettazione_pipeline/muri-rotti-zoom.png">
<figcaption><b>Muri</b><span>La geometria appare rotta</span></figcaption>
</figure>
<figure>
<img src="figures/3_progettazione_pipeline/finestre-rumorose.png">
<figcaption><b>Finestre</b><span>Una finestra può contenere fino a un centinaio di vertici</span></figcaption>
</figure>
</div>

---

<style scoped>
.row { grid-template-columns: 1fr auto 1fr; gap: 16px; }
section { --img-h: 380px; --cap: 0.85em; }
.arrow { font-size: 2.4em; }
</style>

<div class="row">
<figure>
<img src="figures/3_progettazione_pipeline/caso-studio.png">
<figcaption>Prima</figcaption>
</figure>
<div class="arrow">&rarr;</div>
<figure>
<img src="figures/3_progettazione_pipeline/modello-cad-finale.png">
<figcaption>Dopo: layer e geometria corretti</figcaption>
</figure>
</div>

---

<!-- 
Il formato DXF organizza il disegno in primitive geometriche: LINE, POLYLINE, ARC, CIRCLE, ecc. Queste entità sono raggruppate in layer e blocchi (componenti riutilizzabili comunemente impiegati per rappresentare porte e finestre).

L’obiettivo della fase di parsing è estrarre da queste primitive le informazioni geometriche necessarie alla costruzione della mesh. Siccome sappiamo come sono strutturati i nomi dei layer/blocchi ci viene molto più facile estrarre le primitive collassando tutte le primitive sotto forma di segmenti, così da ottenere una rappresentazione uniforme.

Un segmento è semplicemente un coppia di vertici a cui viene associato un TAG che ne indica la semantica (muro, porta o finestra).

Quasi sempre, le primitive più uilizzate nei disegni sono: LINE (coppia di vertici), POLYLINE (contorno chiuso) e ARC. Questi ultimi sono tipici delle porte: si estrae il raggio, il punto base, l’angolo iniziale e finale e a partire da questi dati si calcolano i due punti estremi.
-->

<style scoped>
.row { grid-template-columns: 1fr 1.3fr; align-items: center; }
section { --img-h: 470px; font-size: 20px; }
.col { gap: 10px; }
</style>

# Passo 1: Parsing

<div class="row">
<div class="col">

<div class="card">

## Primitive DXF

`LINE`, `POLYLINE`, `ARC`, `CIRCLE`... raggruppate in **layer** e **blocchi** (componenti riutilizzabili, usati per porte e finestre)

</div>

<div class="card orange">

## Obiettivo

Estrarre muri, porte e finestre, collassando tutte le primitive sottoforma di **segmento**:
$S = <\mathbf{p_0, p_1,} \text{TAG}>$
$TAG = \{ \text{WALL, DOOR, WINDOW} \}$


</div>

<div class="card green">

## Risultato

**3 insiemi distinti** di segmenti: muri, porte, finestre

</div>

</div>

<figure>
<img src="figures/3_progettazione_pipeline/parsing.png">
<figcaption>Risultato del parsing</figcaption>
</figure>
</div>


---

<!-- 
Nei disegni CAD, in uno stesso punto ci sono più vertici (appartenenti a segmenti diversi) sovrapposti oppure molto vicini da essere indistinguibili. Questa operazione serve sia a ridurre il numero di vertici da elaborare ma soprattutto per riuscire a rappresentare la geometria come grafo con vertici univoci.

L’obiettivo è prendere l’insieme dei segmenti e rimuovere vertici che sono considerati in più, ovvero che distano meno di $epsilon$. Per risolvere questo problema in modo efficiente si è scelto di implementare una struttura dati spaziale efficiente: un hash spaziale.
Il funzionamento è semplice: lo spazio 2D viene suddiviso in una griglia regolare, in cui ogni cella ha una dimensione fissa. A ciascun punto viene associata esattamente una cella, calcolata in base alle sue coordinate (posizione).
Una proprietà interessante di questa struttura è che se la dimensione della cella è scelta pari al raggio di ricerca 𝑟, due punti che distano meno di 𝑟 si trovano necessariamente nella stessa cella oppure in una delle celle immediatamente adiacenti.
Questo significa che, per rispondere a un’interrogazione centrata in un punto 𝑝, non è necessario esaminare l’intero insieme di punti, ma è sufficiente fare una ricerca nella griglia 3x3

Il parametro epsilon definisce il raggio di ricerca 𝑟 e di conseguenza la dimensione della cella. Dato un punto 𝑝, cerca nella griglia 3x3 un vertice già esistente che disti meno di epsilon; se esiste già il nuovo vertice viene scartato senza essere inserito nella griglia, altrimenti viene inserito.
In questo caso è stato utilizzato epsilon=10^(-2) per avere una riduzione di circa il 50% dei vertici. In generale non esiste un valore unico per tutti i modelli, occorre provare e verificare.

Durante la creazione della struttura spaziale prendo ogni singolo segmento (dei muri) e rimuovo i duplicati, ma allo stesso tempo mi salvo gli archi Edge con riferimenti ai due vertici estremi. Alla fine io otterrò sia l'insieme dei vertici univoci V, sia anche la lista degli archi E. Praticamente un grafo.
-->

<style scoped>
section { --img-h: 270px; --cap: 0.8em; font-size: 20px; }
.row { grid-template-columns: repeat(3, 1fr); gap: 16px; align-items: start; }
.row + .row { grid-template-columns: 1fr 1fr; margin-top: 14px; align-items: stretch; }
</style>

# Passo 2: Collasso dei vertici

<div class="row">
<figure>
<img src="figures/3_progettazione_pipeline/wall_segments.jpg">
<figcaption>Input: segmenti dei muri</figcaption>
</figure>
<figure>
<img src="figures/3_progettazione_pipeline/before_snap.jpg">
<figcaption><b>Prima</b>176 vertici</figcaption>
</figure>
<figure>
<img src="figures/3_progettazione_pipeline/after_snap.jpg">
<figcaption><b>Dopo</b>88 vertici (50% in meno), SSIM: 0.999</figcaption>
</figure>
</div>

<div class="row">
<div class="card red">

## Problema

Vertici **duplicati**, sovrapposti o così vicini da essere indistinguibili

</div>
<div class="card green">

## Obiettivi

1. **Ridurre** il numero di vertici da elaborare
2. Rappresentare la geometria come **grafo** con vertici univoci

</div>
</div>

---

<style scoped>
.row { grid-template-columns: 1.1fr 1fr; align-items: center; }
section { --img-h: 420px; font-size: 20px; }
.col { gap: 8px; }
figure video { width: 100%; height: var(--img-h); object-fit: contain; }
</style>

# Hash spaziale

<div class="row">

<figure>
<video src="figures/recording-spatial-hash.mp4" autoplay loop muted playsinline></video>
</figure>

<div class="col">

<div class="card">

## Griglia

Spazio 2D diviso in **celle di lato fisso**: ogni punto cade in una cella, in base alle sue coordinate

</div>

<div class="card orange">

## Ricerca 3×3

Con cella pari al raggio $r$, due punti a distanza $< r$ stanno in una cella adiacente: basta cercare nelle **3×3 celle**

</div>

<div class="card red">

## Regola

$\varepsilon$ = raggio di ricerca = lato della cella. Se esiste un vertice a distanza $< \varepsilon$ il nuovo viene **scartato**, altrimenti **inserito**. Qui $\varepsilon = 10^{-2}$ (da tarare per ogni disegno)

</div>

<div class="card green">

## Risultato

Vertici univoci $V$ e archi $E$: un **grafo non connesso** $G = \langle V, E \rangle$

</div>

</div>
</div>

---

<!-- 
Ricostruzione dei varchi

1. Ricostruzione porte

L'idea è di iterare su tutti i segmenti (rossi) delle porte, prelevare le due estremità (vertici) e, sfruttando la struttura spaziale, cercare i due vertici di muro più vicini che rappresentano i punti di giunzione. Una volta recuperati i due vertici di giunzione inseriamo un nuovo Edge (con tag:DOOR).

Per capire meglio osserviamo la seguente immagine:  Grafo dei muri e nuovo arco (tag DOOR) tra i vertici 𝐶 e 𝐷, individuati come i vertici dei muri più vicini alle estremità del segmento della porta.
<figures/3_progettazione_pipeline/wall_graph.png>

Inserire un solo segmento non ci basta. Per ottenere un poligono semplice è necessario inserire un secondo arco (parallelo) tra i vertici 𝐴,𝐵. 
Il criterio per individuare 𝐴 e B: conosciamo i due vertici di giunzione C,D e calcoliamo il versore CD. Consideriamo il vertice C e i suoi due vicini che sono F,A. Dobbiamo verificare quale dei due vertici il vettore risultante sia ortogonale a CD. Si sfrutta la proprietà del prodotto scalare tra due vettori unitari: se il risultato è 0 i due vettori sono perpendicolari; altrimenti se è -1 sono opposti.
In questo caso il vertice candidato è A:
- CD dot CF = -1 
- CD dot CA = 0

Dall'altra parte il vertice candidato è B:
- CD dot DH = -1 
- CD dot DB = 0

Inseriamo il secondo Edge tra A,B.
Figura: inserimento del secondo edge.
<figures/3_progettazione_pipeline/wall_graph2.png>

2. Ricostruzione finestre
In questo caso ogni finestra contiene decine di segmenti e vertici. In altri modelli possono essere rappresentate in maniera più o meno diversa. è necessario un approccio più generale basato sul clustering spaziale. 
L’obbiettivo è di raggruppare dei punti in gruppi (cluster). Un parametro fondamentale è il raggio di prossimità `r`: per ogni punto viene considerato un piccolo cerchio di raggio r, si contano quanti punti cadono dentro al cerchio e se il numero è maggiore di `m` allora è un "core point". In altre parole, se in un intorno di un punto p sono presenti almeno 2 punti allora si dice essere core point e viene assegnato ad un certo cluster.  I punti che si trovano vicino al core point vengono considerati dentro al cluster. è importante impostare bene il raggio `r` perché troppo grande tende a far collassare cluster distinti in un unico
cluster; al contrario, un valore troppo piccolo produce una frammentazione eccessiva.
<3_dbscan.png> -> <7_dbscan.png>

Per ogni finestra viene calcolato un cluster indipendente. Qua il valore `m` è di default 2, mentre deve essere tarato il valore `r` (diverso da disegno a disegno). 
Per il clustering vengono generati nuovi punti campionati dai singoli segmenti di finestre e su questi nuovi punti si esegue l'algoritmo di clustering.

Dopo che abbiamo generato i cluster, calcoliamo per ognuno il bounding box da cui si estrae uno dei due lati lunghi, che sarà il segmento che collega i due vertici dei muri. In questo modo si applica lo stesso ragionamento e procedura appena vista per le porte. L’unica differenza rispetto al caso delle porte è che gli edge vengono taggati WINDOW anziché DOOR.
<cluster.jpg>

Importante: alla fine della ricostruzione non sono stati creati nuovi vertici, ma sono solo stati aggiunti nuovi edge.

Figura: Rappresentazione finale
<doors-windows-connected.jpg>
-->

<style scoped>
.row { grid-template-columns: 1.2fr 1fr; align-items: center; }
section { --img-h: 400px; font-size: 20px; }
.col { gap: 10px; }
</style>

# Passo 3: Ricostruzione porte

<div class="row">
<figure>
<img src="figures/3_progettazione_pipeline/wall_graph.png">
<figcaption>Nuovo arco (tag <code>DOOR</code>) tra C e D, i vertici dei muri più vicini agli estremi della porta</figcaption>
</figure>

<div class="col">

<div class="card red">

## Problema

Il grafo appena creato contiene solo i **muri**. I segmenti delle porte collegano due vertici di muro, ma non sappiamo quali

</div>

<div class="card">

## Idea

Per ogni segmento rosso (porta) prendo i due estremi e, con la struttura spaziale, cerco i vertici di muro più vicini: i punti di giunzione ($C$, $D$)

</div>

<div class="card green">

## Risultato

Nuovo edge $e = \langle C, D, \text{DOOR} \rangle$

</div>

</div>
</div>

---

<style scoped>
.row { grid-template-columns: 1.2fr 1fr; align-items: center; }
section { --img-h: 400px; font-size: 20px; }
.col { gap: 10px; }
</style>

# Passo 3: Ricostruzione porte (2)

<div class="row">
<figure>
<img src="figures/3_progettazione_pipeline/wall_graph_2.png">
<figcaption>Inserimento del secondo edge tra A e B</figcaption>
</figure>

<div class="col">

<div class="card red">

## Problema

Un solo edge non basta per ottenere un **poligono chiuso**: serve un secondo edge tra $A$ e $B$

</div>

<div class="card">

## Come trovare A e B

Calcolo il versore $\vec{CD}$. Tra i vicini di $C$ ($F$, $A$) scelgo quello **ortogonale** a $\vec{CD}$, sfruttando il prodotto scalare tra versori

</div>

<div class="card orange">

## In pratica

$\vec{CD}\cdot\vec{CF} = -1$, $\ \vec{CD}\cdot\vec{CA} = 0 \to$ **A**

$\vec{CD}\cdot\vec{DH} = -1$, $\ \vec{CD}\cdot\vec{DB} = 0 \to$ **B**

</div>

<div class="card green">

## Risultato

Nuovo edge tra $A$ e $B$. Nessun vertice nuovo inserito.

</div>

</div>
</div>

---

<style scoped>
section { --img-h: 250px; --cap: 0.8em; font-size: 20px; }
.row { grid-template-columns: 1fr auto 1fr; gap: 16px; align-items: center; }
.row + .row { grid-template-columns: repeat(3, 1fr); margin-top: 14px; align-items: stretch; }
.arrow { font-size: 2.4em; }
</style>

# Passo 3: Ricostruzione finestre (clustering DBSCAN)

<div class="row">
<figure>
<img src="figures/2_background_teorico/3_dbscan.png">
<figcaption>Punti campionati dai segmenti</figcaption>
</figure>
<div class="arrow">&rarr;</div>
<figure>
<img src="figures/2_background_teorico/7_dbscan.png">
<figcaption>Cluster individuati</figcaption>
</figure>
</div>

<div class="row">
<div class="card">

## Perché

Ogni finestra ha decine di segmenti e cambia da modello a modello: serve un approccio generale

</div>
<div class="card orange">

## Parametri

Raggio $r$ (da tarare per ogni disegno) e $m = 2$: se nell'intorno di $p$ ci sono almeno 2 punti, $p$ è un **core point**

</div>
<div class="card red">

## Attenzione a $r$

**Troppo grande**: cluster distinti si fondono. **Troppo piccolo**: frammentazione

</div>
</div>

---

<style scoped>
section { --img-h: 270px; --cap: 0.8em; font-size: 20px; }
.row { grid-template-columns: 1fr 1fr; gap: 16px; align-items: start; }
.row + .row { margin-top: 14px; align-items: stretch; }
</style>

<div class="row">
<figure>
<img src="figures/3_progettazione_pipeline/clusters.jpg">
<figcaption>Cluster e bounding box di una finestra</figcaption>
</figure>
<figure>
<img src="figures/3_progettazione_pipeline/doors-windows-connected.jpg">
<figcaption>Grafo connesso con porte e finestre ricostruite</figcaption>
</figure>
</div>

<div class="row">
<div class="card">

## Procedura

**Bounding box** del cluster → **lato lungo** come segmento di giunzione → stessa procedura delle porte, con $\text{TAG = WINDOW}$

</div>
<div class="card green">

## Importante

**Nessun nuovo vertice**: sono stati aggiunti solo nuovi edge

</div>
</div>

<!--
Estrazione Facce

Le strutture dati per la connettività derivano dall’idea di rappresentare una mesh come grafo: un insieme di vertici e di archi, a cui si aggiunge un insieme di facce definite come cicli ordinati di vertici o archi.
Un grafo è planare se può essere disegnato senza che i suoi archi si intersechino. Quando gli archi non sono curvi ma sono semplicemente rettilinei si parla di Grafo Planare Rettilineo (Planar Straight-Line Graph): ed è il punto di partenza per operazioni come l’estrazione delle facce racchiuse dal grafo.

Domanda: Non è possibile estrarre le facce usando unicamente quel grafo?
Il grafo da solo non basta. Il grafo contiene vertici ed edge, ma non dice in che ordine gli edge si dispongono attorno a un vertice. Una faccia è un ciclo chiuso che, a ogni vertice, gira sempre dallo stesso lato (ad esempio all’edge successivo in senso antiorario).
`twin` e `next` sono quell’ordinamento già calcolato. Quindi l’Half-Edge non è un’alternativa al grafo, ma lo arricchisce con la connettività.

Oltre alla semplice memorizzazione indicizzata, esistono diverse strutture pensate per rendere esplicita ed efficiente la navigazione della connettività di una mesh. Una di queste è la struttura Half-Edge (o doubly-linked edge list): per ogni edge ci sono due "half-edge" orientati in direzioni opposte, uno per ciascuna delle due facce che condividono l’arco. Ogni half-edge memorizza un riferimento alla faccia sul proprio lato dell’arco, al vertice verso cui punta, al proprio gemello (l’half-edge opposto) e all’half-edge successivo lungo il perimetro della propria faccia. Percorrendo ripetutamente questi riferimenti "successivi" si ottiene un ciclo chiuso corrispondente esattamente al contorno di una faccia.
<cgal-halfedge.png>, <half-edge.png>

L'informazione sul tipo di segmento (WALL, DOOR, WINDOW) viene mantenuta sugli half-edge e ai loro twin, così da essere disponibile per la classificazione delle facce.

Per implementare questa struttura è stata utilizzata la libreria CGAL che fornisce tra le altre la struttura Arrangement che contiene vertici, half-edge orientati e una lista di poligoni (facce). 
-->

---

<style scoped>
.row { grid-template-columns: 1.2fr 1fr; align-items: center; }
section { --img-h: 400px; font-size: 20px; }
.col { gap: 10px; }
</style>

# Passo 4: Estrazione delle facce

<div class="row">
<figure>
<img src="figures/3_progettazione_pipeline/doors-windows-connected.jpg">
<figcaption>Grafo dopo la ricostruzione dei varchi</figcaption>
</figure>

<div class="col">

<div class="card">

## Input

Grafo planare con vertici univoci ed edge con TAG (`WALL`, `DOOR`, `WINDOW`)

</div>

<div class="card green">

## Obiettivo

Estrarre le facce: cicli chiusi di edge, cioè **poligoni 2D**

</div>

<div class="card red">

## Problema

Il grafo dice chi è collegato a chi, ma non l'ordine degli edge attorno a un vertice: serve una struttura che renda esplicita la **connettività**

</div>

</div>
</div>

---

<style scoped>
section { --img-h: 260px; --cap: 0.8em; font-size: 20px; }
.row { grid-template-columns: 1fr 1fr; gap: 16px; align-items: start; }
.row + .row { grid-template-columns: repeat(3, 1fr); margin-top: 14px; align-items: stretch; }
</style>

# La struttura Half-Edge

<div class="row">
<figure>
<img src="figures/2_background_teorico/half-edge.png">
<figcaption>Due half-edge opposti per ogni edge</figcaption>
</figure>
<figure>
<img src="figures/2_background_teorico/cgal-halfedge.png">
<figcaption>Half-edge in CGAL</figcaption>
</figure>
</div>

<div class="row">
<div class="card">

## Ogni half-edge conosce

La **faccia** sul suo lato, il **vertice** a cui punta, il **twin** (opposto) e il **next** lungo la faccia

</div>
<div class="card orange">

## Percorrendo i next

Si ottiene un **ciclo chiuso**: il contorno di una faccia

</div>
<div class="card green">

## CGAL Arrangement

Vertici, half-edge e **lista di facce**. I **tag** restano su ogni half-edge e sul suo twin. Risultato: **poligoni 2D**

</div>
</div>

---

<!-- 
Classificazione Facce

Classificare le facce è NECESSARIO per effettura l'estrusione differenziata. Itero su tutte le facce e le classifico in base a quali tipi di segmenti è costituita: ad esempio posso considerare una faccia DOOR se è costituita esattamente da 2 edge rossi (DOOR) e 2 edge neri (muri), una faccia WINDOW se è costituita esattamente da 2 edge blu (WINDOW) e 2 edge neri, oppure una faccia DOOR se è costituita da tutti segmenti neri. 
<faces.png>
-->

<style scoped>
.row { grid-template-columns: 1.2fr 1fr; align-items: center; }
section { --img-h: 420px; font-size: 20px; }
.col { gap: 10px; }
</style>

# Classificazione delle facce

<div class="row">
<figure>
<img src="figures/3_progettazione_pipeline/faces.png">
<figcaption>Facce classificate: stanze, porte e finestre</figcaption>
</figure>

<div class="col">

<div class="card">

## Perché

Serve per l'**estrusione differenziata**: ogni tipo di faccia ha un'altezza diversa

</div>

<div class="card red">

## Porta

2 edge `DOOR` + 2 edge `WALL`

</div>

<div class="card orange">

## Finestra

2 edge `WINDOW` + 2 edge `WALL`

</div>

<div class="card green">

## Muri

Solo edge `WALL`

</div>

</div>
</div>

---

<!-- 
Estrusione 

L’estrusione è il processo di trasformare forme 2D a forme tridimensionali assegnando un'altezza e una direzione. Consiste nel trasformare poligoni in solidi lungo un asse.
Siccome noi non abbiamo a che fare con forme particolari o forme curve, il procedimento è molto semplice: si duplicano i vertici del piano 2d, si traslano ad una certa altezza e per ogni lato si crea un quad verticale (definito da 2 triangoli).
<Processo_di_estrusione_da_2D_a_3D.png>, <estrusione_poligono_muro_CAD.gif>
-->

<style scoped>
.row { grid-template-columns: 1.4fr 1fr; align-items: center; gap: 24px; }
.col { gap: 10px; }
section { --img-h: 420px; --cap: 0.8em; }
.col figure img { height: 195px; }
</style>

# Passo 5: Estrusione differenziata

<div class="row">
<figure>
<img src="figures/estrusione_animata.gif">
<figcaption>Da poligono 2D a solido 3D</figcaption>
</figure>

<div class="col">
<figure>
<img src="figures/estrusione_porta.png">
<figcaption>Porta</figcaption>
</figure>
<figure>
<img src="figures/estrusione_finestra.png">
<figcaption>Finestra</figcaption>
</figure>
</div>
</div>

<!-- 
Rappresentazione di una mesh

La maggior parte dei modelli 3D utilizzati in applicazioni reali è composta da triangoli. Una mesh triangolare è una rete di triangoli connessi tramite vertici e archi condivisi a formare un’unica superficie continua. I triangoli possono essere memorizzati come entità indipendenti, ciascuna con i propri tre vertici (duplicazione dei vertici e spreco di spazio), oppure memorizzare una sola volta ciascun vertice distinto e far sì che ogni triangolo vi faccia riferimento tramite un indice (indexed triangle mesh). Un vertice può essere arricchito con attributi aggiuntivi: normale e texture coordinate che sono fondamentali durante i calcoli di shading (illuminazione)
Figura: rappresentazione di una indexed triangle mesh
<indexed-triangle-mesh.png>

Algoritmi di triangolazione
Un problema classico della computer grafica è quello di scomporre un poligono semplice in triangoli. La scomposizione di un poligono semplice in triangoli è chiamata triangolazione del poligono. Qua è stato considerato Constrained Delaunay Triangulation. 
Il flusso classico per costruire un CDT:
1. triangolazione iniziale (utilizzando ad es. Ear Clipping o Sweep Line Algorithm). Nel mio caso con Poly2tri utilizza Sweep Line.
2. si applica l'algoritmo di Flip di Lawson solo sugli archi interni che violano la condizione di Delaunay.

La condizione di Delaunay per un insieme di punti: nessun punto deve trovarsi all'interno del cerchio circoscritto di un triangolo. Il centro di questo cerchio è chiamato centro del cerchio circoscritto al triangolo ed è equidistante da tutti i vertici.

CDT presenta due caratteristiche importanti: evita la formazione di triangoli degeneri; consente di specificare esplicitamente quali segmenti debbano necessariamente far parte della triangolazione risultante.

Dal punto di vista della complessità computazionale, nel caso in cui l’input sia un singolo poligono semplice, la CDT può essere calcolata in tempo lineare O(𝑛).
<recording_DT.webm>
-->

---

<style scoped>
.row { grid-template-columns: 1.2fr 1fr; align-items: center; }
section { --img-h: 400px; font-size: 20px; }
.col { gap: 10px; }
</style>

# Rappresentazione di una mesh

<div class="row">
<figure>
<img src="figures/2_background_teorico/indexed-triangle-mesh.png">
<figcaption>Indexed triangle mesh: buffer dei vertici e buffer degli indici</figcaption>
</figure>

<div class="col">

<div class="card">

## Mesh triangolare

Rete di triangoli connessi, che non deve contenere buchi

</div>

<div class="card orange">

## Vertici

Salvati in un **vertex buffer**.

Per ogni vertice vengono calcolati e memorizzati i seguente attributi: posizione, normale, texture coordinate.

</div>

<div class="card green">

## Indici

Ogni triangolo richiama i suoi vertici gli indici.

L'insieme degli indici sono salvati in un **index buffer**.

</div>

</div>
</div>

---


<style scoped>
.row { grid-template-columns: 1.1fr 1fr; align-items: center; }
section { --img-h: 420px; font-size: 19px; }
.col { gap: 8px; }
figure video { width: 100%; height: var(--img-h); object-fit: contain; }
</style>

# Passo 6: Triangolazione di Delaunay

<div class="row">
<figure>
<video src="figures/recording_DT.mp4" autoplay loop muted playsinline></video>
<figcaption>Flip degli archi che violano la condizione di Delaunay</figcaption>
</figure>

<div class="col">

<div class="card red">

## Problema

Scomporre un poligono semplice in triangoli, usando l'algoritmo **Constrained Delaunay Triangulation**

</div>

<div class="card">

## Flusso

1. **Triangolazione iniziale** (Ear Clipping, Sweep Line). Qui: Poly2Tri con Sweep Line
2. **Flip di Lawson** solo sugli archi interni che violano la condizione di Delaunay

</div>

<div class="card orange">

## Condizione di Delaunay

Nessun punto dentro il cerchio circoscritto di un triangolo

</div>

<div class="card green">

## Vantaggi

Evita i **triangoli degeneri** e permette di imporre segmenti obbligatori

</div>

</div>
</div>

---

<style scoped>
section { --img-h: 480px; --cap: 0.8em; }
figure video { width: 100%; height: var(--img-h); object-fit: contain; }
</style>

# Mesh Preview

<figure>
  <video src="figures/mesh-preview.mp4" autoplay loop muted playsinline></video>
<figcaption>Mesh 3D generata dalla pipeline (prima dell'inserimento degli asset)</figcaption>
</figure>

<!--
Esportazione

Adesso abbiamo una mesh tridimensionale effettiva  con vertici, indici, normali e texture coordinates. La mesh deve essere esportata in formato aperto (GLTF).

Se provassi a renderizzare la mesh così com'è vedrei dei semplici buchi vuoti a ridosso di porte e finestre. Non è ancora accettabile. Vogliamo la possibilità di potere inserire degli asset 3D.

Una prima ipotesi era inserire gli asset direttamente nel processo di generazione della mesh, ma questo approccio risulta poco conveniente. Consideriamo solo il caso delle porte. Un asset di una porta dettagliata può arrivare a contenere centinaia/migliaia di vertici. Se in un appartamento sono presenti 10 porte, la geometria della porta verrebbe duplicata 10 volte, aumentando notevolmente la dimensione del file. Oppure, Inoltre, se si vuole cambiare asset oppure modificare l’orientamento di una porta bisognerebbe rieseguire l’intera pipeline. Occorre anche preservare i materiali originali dell’asset scaricato, complicando ulteriormente il processo.

La soluzione più flessibile e mantenibile consiste nell’adottare un approccio a placeholder, spostando il caricamento degli asset direttamente in Blender. Blender gestisce l'instancing: consideriamo una una mesh di una porta ripetuta tante volte, i vertici della stessa mesh non vengono duplicati tante volte quante il numero di porte. I vertici sono allocati una volta sola e condivisi tra le altre instance. Quello che cambia è la matrice di trasformazione (traslazione, scala, rotazione) e il materiale (è possibile anche cambiare materiale alle diverse porte)

Mi serve solamente esportare informazioni su dove inserire gli asset, con quale orientamento e con quale scala. Tutte queste informazioni vengono infine serializzate ed esportate sotto forma di file JSON, insieme al modello GLTF.

In pratica la pipeline esporta due file:
- mesh GLTF
- placeholders JSON
-->

---

<style scoped>
section { font-size: 19px; }
h1 { margin-bottom: 14px; }
.row { grid-template-columns: 1fr 1fr; align-items: stretch; gap: 12px 20px; }
.row + .row { margin-top: 12px; }
.flow { display: grid; grid-template-columns: 1fr auto 1fr auto 1fr; gap: 10px; align-items: center; margin-top: 12px; }
.flow .card { text-align: center; }
.flow .card h2 { margin-bottom: 2px; }
</style>

# Passo 7: Esportazione

<div class="row">
<div class="card orange">

## Problema

La mesh esportata in **GLTF** ha dei buchi vuoti a ridosso di porte e finestre: servono asset 3D

</div>
<div class="card">

## Instancing in Blender

Una mesh ripetuta non duplica i vertici: sono allocati **una sola volta** e condivisi. Cambiano solo la matrice di trasformazione e il materiale

</div>
</div>

<div class="row">
<div class="card red">

## Asset nella pipeline (scartato)

- Geometria **duplicata**: 10 porte, 10 copie
- Cambiare asset o orientamento = **rieseguire la pipeline**
- **Materiali originali** da preservare

</div>
<div class="card green">

## Approccio a placeholder (scelto)

La pipeline esporta solo dove inserire l'asset (`center`), l'**orientamento** (`rotate_z`) e la **scala**. Il caricamento avviene in Blender

</div>
</div>

<div class="flow">
<div class="card">

## Pipeline

genera la mesh

</div>
<div class="arrow">&rarr;</div>
<div class="card orange">

## `mesh.gltf`
## `placeholders.json`

</div>
<div class="arrow">&rarr;</div>
<div class="card green">

## Blender

carica gli asset

</div>
</div>

--- 

<!--

Rendering

Per la fase di rendering viene utilizzato Blender, sia tramite uno script Python basato sulle API bpy, sia tramite GUI.

Il file GLTF esportato contiene esclusivamente la geometria
della mesh (posizioni, normali e coordinate texture) e non include alcuna informazione sui materiali. I materiali, gli asset, l'illuminazione, la camera, l’environment map vengono definiti in Blender. Questa soluzione è anche la più flessibile: è possibile cambiare i materiali dover rieseguire ogni volta l’intera pipeline.

Inizialmente si era valutato di eseguire l’intero rendering all’interno dello script Python. Questo approccio si è tuttavia rivelato poco pratico: posizione e rotazione della camera, così come posizione e intensità delle luci, avrebbero dovuto essere codificate direttamente nel codice (hard-coded), rendendo la scelta dei parametri tediosa e poco intuitiva.

Si è quindi adottato un approccio ibrido: lo script Python automatizza le operazioni ripetitive e deterministiche, mentre la composizione finale della scena (camera e luci) è affidata alla GUI. Nello script:
- importo la mesh GLTF
- definisco i materiali (albedo, normal, roughness, ecc.) diversi per il pavimento e muri
- carico l’environment map
- leggo i placeholder e carico gli asset
- alla fine creo la scena iniziale file .blend

Il file .blend generato viene aperto in Blender tramite GUI e qua possiamo navigare e impostare la camera, posizionare le luci e modificarne le proprietà maniera più semplice. Grazie alla modalità GUI è possibile aggiungere facilmente ulteriori asset per arredarla come un divano, una TV, una lampada, ecc.

Il rendering viene eseguito con Cycles: physically-based path tracing render engine.
Il rendering è stato effettuato a risoluzione 1280x720, con 128 samples per pixel, denoiser (OpenImageDenoise), su CPU (AMD Ryzen 5 2600X) perché la mia GPU non è adatta  per PT con Cycles. 
Il tempo di rendering varia in base alla complessità della scena: da 3 a 7 minuti per una immagine.

Risultati

<rendering-1.png>
<rendering-2.png>
<rendering-3.png>
<rendering-4.png>
-->

<style scoped>
section { font-size: 19px; }
h1 { margin-bottom: 14px; }
.row { grid-template-columns: 1fr 1fr; align-items: stretch; gap: 12px 20px; }
.col { gap: 10px; }
.col .card { flex: 1; }
</style>

# Rendering in Blender: approccio ibrido

<div class="row">
<div class="col">

<div class="card orange">

## GLTF: solo geometria

Posizioni, normali e coordinate texture, nessun materiale. Materiali, asset, luci, camera ed environment map sono definiti in **Blender**: posso cambiare i materiali senza rieseguire la pipeline.

</div>

<div class="card red">

## Solo script Python? Scartato

Camera e luci (posizione, rotazione, intensità) sarebbero **hard-coded**: molto scomodo e poco intuitivo

</div>

</div>
<div class="col">

<div class="card">

## Script Python (`bpy`)

Operazioni ripetitive e deterministiche

1. Importa la **mesh GLTF**
2. Definisce i **materiali** (albedo, normal, roughness...) per pavimento e muri
3. Carica l'**environment map**
4. Legge i **placeholder** e carica gli **asset**
5. Crea la scena iniziale **`.blend`**

</div>

<div class="card green">

## GUI di Blender

Camera e luci impostate navigando la scena, in modo semplice. Aggiungere arredi (divano, TV, lampada) è immediato

</div>

</div>
</div>

---

<style scoped>
section { font-size: 20px; }
.row { grid-template-columns: 1fr 1fr 1fr; align-items: stretch; gap: 16px; margin-top: 20px; }
.row .card { text-align: center; }
.row .card h2 { font-size: 0.9em; }
.big { font-size: 1.5em; font-weight: bold; color: var(--c); margin: 4px 0; }
.sub { text-align: center; margin: 24px 0 0 0; font-size: 0.95em; }
</style>

# Rendering con Cycles

<p class="goal">
  <b>Cycles</b>: path tracer <i>physically-based</i>
</p>

<div class="row">
<div class="card">

## Risoluzione

<p class="big">1280×720</p>

</div>
<div class="card orange">

## Campioni

<p class="big">128 spp</p>

con denoiser **OpenImageDenoise**

</div>
<div class="card green">

## Tempo

<p class="big">3-7 min</p>

per immagine, in base alla complessità

</div>
</div>

<p class="sub">Su <b>CPU</b> AMD Ryzen 5 2600X</p>

---

<style scoped>
section { --img-h: 250px; padding-bottom: 12px; }
.row { grid-template-columns: 1fr 1fr; gap: 8px 16px; align-items: center; }
h1 { margin-bottom: 8px; }
</style>

# Risultati

<div class="row">
<figure><img src="figures/4_rendering/draftperson_HousePlan-rendering-1280x720-128-1.png"></figure>
<figure><img src="figures/4_rendering/draftperson_HousePlan-rendering-1280x720-128-2.png"></figure>
<figure><img src="figures/4_rendering/draftperson_HousePlan-rendering-1280x720-128-3.png"></figure>
<figure><img src="figures/4_rendering/draftperson_HousePlan-rendering-1280x720-128-4.png"></figure>
</div>
