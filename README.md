# Tesi Laurea 2026

L'obiettivo del progetto è creare un programma C++ che prenda come input un modello CAD 2D di una planimetria di una stanza o appartamento e creare una mesh 3D con i seguenti attributi: posizione, normale, tex coordinate. La mesh sarà esporta in formato GLTF. Il modello non deve contenere materiali.

Una volta esportato il file gltf, abbiamo uno script Python che genera la scena in Blender del modello. Qua vengono applicati i materiali, caricati gli asset delle porte e delle finestre, impostata camera, le luci, e altri parametri. Alla fine si procede con il rendering fotorealistico con Cycles.
