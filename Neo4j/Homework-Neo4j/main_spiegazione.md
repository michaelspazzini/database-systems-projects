# Spiegazione `main.ipynb`

Questo file riassume il notebook [`main.ipynb`](/d:/Progetti/SEBD/Neo4j/Homework-Neo4j/main.ipynb) cella per cella, con particolare attenzione alle query Cypher e alle scelte di modellazione del grafo.

## Obiettivo del notebook

Il notebook costruisce e analizza un grafo Neo4j a partire da un dataset di articoli arXiv. I principali elementi del modello sono:

- `Paper`: rappresenta un articolo scientifico
- `Author`: rappresenta un autore
- `Topic`: rappresenta un topic estratto dagli abstract

Le relazioni principali sono:

- `(:Author)-[:AUTHORED]->(:Paper)`
- `(:Paper)-[:ON_TOPIC {probability}]->(:Topic)`
- `(:Author)-[:COAUTHORED_WITH {papers_in_common}]->(:Author)`
- `(:Paper)-[:SIMILAR_TOPIC {score}]->(:Paper)`
- `(:Author)-[:SIMILAR_AUTHOR {score}]->(:Author)`

---

## Cella 1: import e connessione

Questa cella:

- importa le librerie Python
- definisce i path dei file CSV
- crea il driver Neo4j
- verifica la connessione al database

### Librerie usate

- `pandas`: manipolazione tabellare dei dati
- `matplotlib.pyplot`: grafici
- `ast`: conversione di stringhe che rappresentano liste in vere liste Python
- `neo4j.GraphDatabase`: connessione a Neo4j
- `itertools.combinations`: generazione di coppie
- `collections.defaultdict`: dizionari con valori di default

### Oggetti importanti

- `URI`: indirizzo del server Neo4j
- `AUTH`: credenziali
- `driver`: oggetto centrale usato in tutte le celle per aprire sessioni verso Neo4j

### Funzione `check_connection()`

Chiama:

```python
driver.verify_connectivity()
```

Serve solo a verificare che il database sia raggiungibile.

---

## Cella 2: caricamento e preparazione dei dati

Questa cella legge i CSV in DataFrame pandas e prepara i dati per l'importazione nel grafo.

### Funzione `load_csv_to_dataframe`

Legge un file CSV e restituisce un DataFrame. Se il caricamento fallisce, stampa un errore e restituisce `None`.

### Preparazione di `df_papers`

Passaggi:

- legge il CSV dei paper
- tronca il timestamp completo alla sola data con:

```python
df_papers['published'] = df_papers['published'].str[:10]
```

- rinomina la colonna `published` in `date`
- converte la colonna `authors`, che è una stringa del tipo `"['A', 'B']"`, in una vera lista Python usando:

```python
ast.literal_eval
```

### Preparazione di `df_topics_def`

Contiene la definizione dei topic. Le colonne rilevanti sono:

- `Topic`
- `Name`
- `Representation`

### Preparazione di `df_papers_topics`

Contiene per ogni paper:

- il topic principale
- i topic più rilevanti
- le relative probabilità

Le colonne `top_topics` e `top_probabilities` vengono convertite da stringhe a liste Python.

---

## Cella 3: creazione del database Neo4j

Query eseguita:

```cypher
CREATE DATABASE homework IF NOT EXISTS
```

### Cosa fa

- crea un database chiamato `homework`
- se esiste già, non fa nulla

### Perché si usa `database="system"`

In Neo4j i comandi amministrativi come `CREATE DATABASE` vanno eseguiti sul database di sistema.

---

## Cella 4: importazione di `Paper` e `Author`

Questa cella costruisce:

- i nodi `Paper`
- i nodi `Author`
- la relazione `AUTHORED`

### Query principale

```cypher
UNWIND $data AS row
MERGE (p:Paper {url: row.url})
SET p.title = row.title,
    p.abstract = row.abstract,
    p.date = date(row.date)

WITH p, row
UNWIND row.authors AS author_name
MERGE (a:Author {name: author_name})
MERGE (a)-[:AUTHORED]->(p)
```

### Spiegazione riga per riga

`UNWIND $data AS row`

- `$data` è una lista di dizionari passata da Python
- `UNWIND` trasforma la lista in righe
- ogni `row` rappresenta un paper

`MERGE (p:Paper {url: row.url})`

- cerca un nodo `Paper` con quella `url`
- se non esiste, lo crea
- `url` è la chiave unica del paper

`SET ...`

- assegna o aggiorna le proprietà del nodo `Paper`
- `date(row.date)` converte la stringa in una vera data Neo4j

`WITH p, row`

- serve a portare avanti il nodo `p` appena creato e la riga corrente

`UNWIND row.authors AS author_name`

- espande la lista degli autori del paper
- si ottiene un autore per riga

`MERGE (a:Author {name: author_name})`

- crea o riusa il nodo autore

`MERGE (a)-[:AUTHORED]->(p)`

- crea la relazione tra autore e paper

### Vincoli

Prima dell'import vengono creati:

```cypher
CREATE CONSTRAINT paper_url IF NOT EXISTS FOR (p:Paper) REQUIRE p.url IS UNIQUE
CREATE CONSTRAINT author_name IF NOT EXISTS FOR (a:Author) REQUIRE a.name IS UNIQUE
```

Servono a:

- evitare duplicati
- accelerare i `MERGE`

### Nota d'esame

La modellazione dell'autore con chiave `name` è una semplificazione. Due persone diverse con lo stesso nome verrebbero fuse nello stesso nodo.

---

## Cella 5: importazione di `Topic` e relazioni `ON_TOPIC`

Questa cella crea:

- i nodi `Topic`
- le relazioni `(:Paper)-[:ON_TOPIC]->(:Topic)`

### Query per i topic

```cypher
UNWIND $data AS row
MERGE (t:Topic {topic_id: row.topic_id})
SET t.name = row.name,
    t.representation = row.representation
```

### Spiegazione

- `topic_id` è la chiave unica del topic
- `name` è l'etichetta descrittiva del topic
- `representation` contiene i termini rappresentativi

### Query per collegare paper e topic

```cypher
UNWIND $data AS row
MATCH (p:Paper {url: row.url})
UNWIND range(0, size(row.top_topics) - 1) AS i
MATCH (t:Topic {topic_id: row.top_topics[i]})
MERGE (p)-[r:ON_TOPIC]->(t)
SET r.probability = row.top_probabilities[i]
```

### Spiegazione riga per riga

`MATCH (p:Paper {url: row.url})`

- recupera il paper già importato

`UNWIND range(0, size(row.top_topics) - 1) AS i`

- itera sugli indici delle liste `top_topics` e `top_probabilities`
- si usa l'indice per tenere allineati topic e probabilità

`MATCH (t:Topic {topic_id: row.top_topics[i]})`

- recupera il topic associato al paper

`MERGE (p)-[r:ON_TOPIC]->(t)`

- crea la relazione tra paper e topic

`SET r.probability = row.top_probabilities[i]`

- salva sulla relazione il peso del topic nel paper

### Perché la probabilità è sulla relazione

Perché non è una proprietà intrinseca del topic, ma del legame tra uno specifico paper e uno specifico topic.

### Vincolo

```cypher
CREATE CONSTRAINT topic_id IF NOT EXISTS
FOR (t:Topic)
REQUIRE t.topic_id IS UNIQUE
```

---

## Cella 6: costruzione della rete di co-authorship

Questa cella crea le relazioni `COAUTHORED_WITH`.

### Idea

Per ogni paper:

- si prende la lista dei suoi autori
- si generano tutte le coppie di autori del paper
- si incrementa il contatore del numero di paper scritti in comune

### Query

```cypher
UNWIND $authors AS name1
UNWIND $authors AS name2
WITH name1, name2
WHERE name1 < name2
MATCH (a1:Author {name: name1})
MATCH (a2:Author {name: name2})
MERGE (a1)-[r:COAUTHORED_WITH]->(a2)
ON CREATE SET r.papers_in_common = 1
ON MATCH SET r.papers_in_common = r.papers_in_common + 1
```

### Spiegazione

`UNWIND $authors AS name1` e `UNWIND $authors AS name2`

- generano il prodotto cartesiano della lista autori con sé stessa

`WHERE name1 < name2`

- elimina i casi `A-A`
- elimina i doppioni speculari `A-B` e `B-A`

`MERGE (a1)-[r:COAUTHORED_WITH]->(a2)`

- crea una sola relazione per coppia di autori

`ON CREATE SET ...`

- se la relazione non esiste, inizializza il numero di paper condivisi a `1`

`ON MATCH SET ...`

- se esiste già, incrementa il contatore

### Perché si fa paper per paper

Una query globale su tutto il grafo andava in `MemoryPoolOutOfMemoryError`. Questa versione è più sicura perché lavora su un singolo paper alla volta.

---

## Cella 7: calcolo della similarità tra paper

Questa è una delle celle più importanti del notebook.

### Obiettivo

Creare relazioni:

- `(:Paper)-[:SIMILAR_TOPIC {score}]->(:Paper)`

Il punteggio `score` è la similarità di Jaccard pesata, detta anche similarità di Ruzicka.

### Formula implementata

Per due paper `p` e `q`, con distribuzioni di topic:

```text
sim_T(p, q) = sum(min(p_i, q_i)) / sum(max(p_i, q_i))
```

dove:

- `p_i` è la probabilità del topic `i` nel paper `p`
- `q_i` è la probabilità del topic `i` nel paper `q`

### Funzione Python `ruzicka_similarity`

Riceve due dizionari:

```python
{topic_id: probability}
```

e calcola:

- numeratore = somma dei minimi
- denominatore = somma dei massimi

### Ottimizzazione importante

Non confronta tutte le coppie di paper. Sarebbe troppo costoso.

Costruisce invece:

- `topic_to_papers`: per ogni topic, la lista dei paper che lo contengono

Da qui genera solo coppie candidate di paper che condividono almeno un topic.

### Query di scrittura

```cypher
UNWIND $data AS row
MATCH (p1:Paper {url: row.url1})
MATCH (p2:Paper {url: row.url2})
MERGE (p1)-[r:SIMILAR_TOPIC]->(p2)
SET r.score = row.score
```

### Spiegazione

- cerca i due paper tramite `url`
- crea una relazione `SIMILAR_TOPIC`
- salva il valore numerico della similarità

### Nota d'esame

Questa è una forte ottimizzazione:

- non si confrontano tutte le coppie di paper
- si confrontano solo quelle che condividono almeno un topic

Questo è corretto perché, se due paper non condividono alcun topic, la similarità risulta `0`.

---

## Cella 8: top 5 autori più prolifici

### Query

```cypher
MATCH (a:Author)-[:AUTHORED]->(p:Paper)
RETURN a.name, count(p) AS num_papers
ORDER BY num_papers DESC
LIMIT 5
```

### Spiegazione

`MATCH (a:Author)-[:AUTHORED]->(p:Paper)`

- prende tutte le coppie autore-paper

`count(p) AS num_papers`

- conta quanti paper sono collegati a ogni autore

`ORDER BY num_papers DESC`

- ordina dal più prolifico al meno prolifico

`LIMIT 5`

- restituisce solo i primi cinque

### Nota

La dicitura "autori o coautori" è concettualmente ridondante: se un autore è legato a un paper da `AUTHORED`, è già coautore di quel paper insieme agli altri autori.

---

## Cella 9: verifica temporale del dataset

### Query

```cypher
MATCH (p:Paper)
WHERE p.date IS NOT NULL
RETURN min(p.date.year) AS min_year, max(p.date.year) AS max_year, count(p) AS total
```

### Cosa fa

- considera solo i paper con data valorizzata
- estrae:
  - anno minimo
  - anno massimo
  - numero totale di paper

### Perché è utile

Serve a capire se il dataset copre più anni oppure no. Nel tuo caso il dataset contiene solo paper del 2021.

---

## Cella 10: andamento mensile dei paper nel 2021

### Query

```cypher
MATCH (p:Paper)
WHERE p.date IS NOT NULL
RETURN p.date.month AS month, count(p) AS num_papers
ORDER BY month
```

### Spiegazione

- raggruppa i paper per mese
- conta quanti paper ci sono in ogni mese
- ordina i risultati cronologicamente

### Parte Python

La parte Python:

- trasforma il risultato in DataFrame
- mappa i numeri dei mesi in nomi brevi
- disegna un grafico a barre
- aggiunge sopra ogni barra il valore numerico

### Nota d'esame

Non è stato fatto il grafico annuale perché il dataset contiene solo il 2021. Il grafico mensile è quindi la rappresentazione temporale più sensata.

---

## Cella 11: similarità tra autori

Questa è la parte più delicata del notebook.

### Obiettivo

Creare relazioni:

- `(:Author)-[:SIMILAR_AUTHOR {score}]->(:Author)`

La formula usata è:

```text
Sim(a1, a2) = 1 / (|P1| * |P2|) * somma di sim_T(pi, pj)
```

dove:

- `P1` è l'insieme dei paper di `a1`
- `P2` è l'insieme dei paper di `a2`
- `sim_T(pi, pj)` è la similarità tra i due paper

### Funzioni di supporto

#### `get_author_papers`

Query:

```cypher
MATCH (a:Author)-[:AUTHORED]->(p:Paper)
RETURN a.name AS author, collect(p.url) AS papers
```

Crea un dizionario Python:

```python
{autore: [lista_url_paper]}
```

#### `get_paper_similarities`

Query:

```cypher
MATCH (p1:Paper)-[r:SIMILAR_TOPIC]->(p2:Paper)
RETURN p1.url AS url1, p2.url AS url2, r.score AS score
```

Costruisce una mappa Python:

```python
{(url1, url2): score}
```

La mappa viene resa simmetrica salvando anche `(url2, url1)`.

### Funzione `compute_author_similarity`

Per due autori:

- scorre tutte le coppie di paper `p1` in `P1` e `p2` in `P2`
- se `p1 == p2`, aggiunge `1.0`
- altrimenti usa `paper_sim_map.get((p1, p2), 0.0)`
- divide infine per `len(P1) * len(P2)`

### Query di scrittura

```cypher
UNWIND $rows AS row
MATCH (a1:Author {name: row.author1})
MATCH (a2:Author {name: row.author2})
MERGE (a1)-[r:SIMILAR_AUTHOR]->(a2)
SET r.score = row.similarity
```

### Spiegazione

- riceve da Python coppie di autori con il punteggio già calcolato
- crea la relazione `SIMILAR_AUTHOR`
- salva il valore `score`

### Limite importante

Nel notebook attuale le coppie di autori vengono generate con:

```python
combinations(authors, 2)
```

Quindi si considerano tutte le coppie possibili di autori. Con oltre 37 mila autori questo porta a un costo di ordine `O(n^2)`, molto elevato. È formalmente corretto, ma molto costoso.

### Possibile domanda d'esame

Perché questo passaggio è così pesante?

Perché:

- il numero di coppie di autori cresce quadraticamente
- per ogni coppia si confrontano anche tutti i paper del primo autore con tutti i paper del secondo

È la parte più costosa di tutto il notebook.

---

## Cella 12: statistiche finali del database

Questa cella stampa un riepilogo finale di nodi e relazioni presenti nel grafo.

### Query eseguite

```cypher
MATCH (n:Topic) RETURN count(n) AS count
MATCH (n:Paper) RETURN count(n) AS count
MATCH (n:Author) RETURN count(n) AS count
MATCH ()-[r:AUTHORED]->() RETURN count(r) AS count
MATCH ()-[r:ON_TOPIC]->() RETURN count(r) AS count
MATCH ()-[r:COAUTHORED_WITH]->() RETURN count(r) AS count
MATCH ()-[r:SIMILAR_TOPIC]->() RETURN count(r) AS count
MATCH ()-[r:SIMILAR_AUTHOR]->() RETURN count(r) AS count
```

### A cosa servono

Servono a verificare che:

- i nodi siano stati creati
- le relazioni siano state create
- il database contenga le quantità attese

### Nota

Se una relazione non esiste ancora, la relativa query può restituire `0`.

---

## Schema logico complessivo

L'ordine generale del notebook è:

1. connessione a Neo4j
2. caricamento e preparazione dei dati
3. creazione del database `homework`
4. importazione di `Paper` e `Author`
5. importazione di `Topic` e delle relazioni `ON_TOPIC`
6. costruzione della rete `COAUTHORED_WITH`
7. calcolo della similarità tra paper `SIMILAR_TOPIC`
8. interrogazioni di analisi sul grafo
9. analisi temporale delle pubblicazioni
10. calcolo della similarità tra autori `SIMILAR_AUTHOR`
11. verifica finale con statistiche aggregate

---

## Osservazioni finali

I punti più forti del notebook sono:

- modellazione chiara del dominio
- uso corretto di `MERGE`, `UNWIND` e dei vincoli
- uso delle probabilità per rappresentare in modo pesato i topic
- calcolo delle similarità a livello di paper e autore

Il punto più costoso è la similarità tra autori, che andrebbe idealmente ottimizzata limitando le coppie candidate.
