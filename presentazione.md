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
<video src="figures/recording_SpatialHash.mp4" autoplay loop muted playsinline></video>
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
