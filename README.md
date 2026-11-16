# Projet Dana | Groupe CAZ, OULAI, ATTEYEH ABI

## **1. Première partie**

### 1.0 Préambule / Installation des outils

Installation d'Apache Jena Fuseki pour héberger et interroger les données RDF via des requêtes SPARQL. 
Utilisation de tarql pour convertir des données tabulaires .CSV en données RDF à l'aide de requêtes SPARQL.
L'IA (ChatGPT, [Claude Opus 5.5](https://claude.com/) a pu être utilisée ponctuellement comme outil d'assistance afin de débloquer certaines situations, notamment pour des rappels de syntaxe Python et PowerShell, ou des tournures de phrase.

### 1.1 Conversion JSON en CSV

Conversion du fichier artists.json (de base) en un fichier CSV exploitable par tarsQL à l'aide du PowerShell Windows :

```PowerShell
Get-Content -Path "artists.json" -Encoding UTF8 | ConvertFrom-Json | ForEach-Object { $_ } | Export-Csv -Path "artistsRaw.csv" -NoTypeInformation -Encoding UTF8 
```

### 1.2 Nettoyage des .CSV à l'aide de Python (Pandas)

Une fois le fichier des artistes converti, on procède à un nettoyage à l'aide d'un script python et de la librairie Pandas : 
- Suppresion de la colonne "artistBio" dont le contenu est un double d'autres colonnes.
- Suppression des lignes dont une ou plusieurs colonnes sont vides.
- Suppression des lignes où les colonnes "BeginDate" et "EndDate" ont une valeur égale à 0.
- Renommage de la colonne "Wiki QID" en "Wiki_QID" pour supprimer l'espace.

```Python
import pandas as pd

data = pd.read_csv("artistsRaw.csv", encoding="utf-8").drop(columns="ArtistBio").dropna()
data = data[data["BeginDate"] != 0]
data = data[data["EndDate"] != 0]
data = data.rename(columns={"Wiki QID": "Wiki_QID"})
data.to_csv("artistsCleaned.csv")
```

Pour la partie oeuvres d'art, on procède de cette façon (étape réalisée avec [Claude Opus 5.5](https://claude.com/) pour la syntaxe Python) :
- Suppression des colonnes artiste du fichier (Gender, Nationality, BeginDate…) car elles sont déjà disponibles dans le fichier des artistes
- Création d'une table de liason entre artistes et oeuvres d'art via la propriété WIKI_QID

## **2. Seconde partie**

### 2.0 Choix du vocabulaire

Au regard de notre jeu de données, nous avons décidé de créer plusieurs artistes de type "Personne", avec comme attributs:

- `foaf:name` et `foaf:gender` pour les colonnes `nom` et `genre`.
- `schema:birthDate`, `schema:deathDate`, `schema:nationality` pour représenter respectivement les colonnes `BeginDate`, `EndDate` et `Nationality`.
- Enfin `owl:sameAs` pour faire le lien avec la colonne `ULAN`.


Ci-dessous le script de création sparql :
```
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX schema: <https://schema.org/>
PREFIX owl: <http://www.w3.org/2002/07/owl#>

CONSTRUCT {
    ?artiste a schema:Person;
            foaf:name ?nom;
            foaf:gender ?genre;
            schema:birthDate ?dateDeNaissance;
            schema:deathDate ?dateDeMort;
            schema:nationality ?nationalite.
}
WHERE {
    BIND(IRI(CONCAT("http://www.wikidata.org/entity/", STR(?Wiki_QID))) AS ?artiste)
    BIND(STR(?DisplayName) AS ?nom)
    BIND(STR(?Gender) AS ?genre)
    BIND(STR(?BeginDate) AS ?dateDeNaissance)
    BIND(STR(?EndDate) AS ?dateDeMort)    
    BIND(STR(?Nationality) AS ?nationalite)
}
```

Nous avons par la suite créé les oeuvres d'arts sous le type VisualArtwork, avec les attributs suivants :
- `schema:name` et `schema:dateCreated` pour les colonnes `titre` et `annee`.
- `schema:artMedium` et `schema:artform` pour le `medium` et la `classification`.
- `schema:height` et `schema:width` représentant les colonnes `hauteur` et `largeur`.

```
PREFIX schema: <https://schema.org/>
PREFIX xsd:    <http://www.w3.org/2001/XMLSchema#>

CONSTRUCT {
  ?oeuvre a schema:VisualArtwork ;
      schema:name ?titre ;
      schema:dateCreated ?annee ;
      schema:artMedium ?medium ;
      schema:artform ?classification ;
      schema:height ?hauteur ;
      schema:width ?largeur .
}
WHERE {
  BIND(IRI(CONCAT("https://www.moma.org/collection/works/", STR(?ObjectID))) AS ?oeuvre)
  BIND(STR(?Title) AS ?titre)
  BIND(xsd:integer(?YearStart) AS ?annee)
  BIND(STR(?Medium) AS ?medium)
  BIND(STR(?Classification) AS ?classification)
  BIND(xsd:decimal(?HeightCm) AS ?hauteur)
  BIND(xsd:decimal(?WidthCm) AS ?largeur)
}
```

Enfin nous créons le mapping de la table de liaison, dont les IRI des artists sont construits comme pour le fichier des artistes

```
PREFIX schema: <https://schema.org/>

CONSTRUCT { ?oeuvre schema:creator ?artiste . }
WHERE {
  FILTER(BOUND(?Wiki_QID) && ?Wiki_QID != "")
  BIND(IRI(CONCAT("https://www.moma.org/collection/works/", STR(?ObjectID))) AS ?oeuvre)
  BIND(IRI(CONCAT("http://www.wikidata.org/entity/", STR(?Wiki_QID))) AS ?artiste)
}
```

### 2.1 Requêtes "intéressantes"

La première requêtes intéressante consiste à voir la répartion des nationalités au sein du jeu de données :

```turtle
PREFIX schema: <https://schema.org/>

SELECT ?nationalite (COUNT(DISTINCT ?artiste) AS ?nbArtistes)
WHERE {
    ?artiste schema:nationality ?nationalite.
}
GROUP BY ?nationalite ORDER BY DESC(?nbArtistes) LIMIT 10
```

La deuxième requête intéressante consiste à analyser la parité entre les artistes : 

```
PREFIX foaf: <http://xmlns.com/foaf/0.1/>

SELECT ?genre (COUNT(DISTINCT ?artiste) AS ?nombreArtistes)
WHERE {
    ?artiste foaf:name ?nom .
    ?artiste foaf:gender ?genre .
}
GROUP BY ?genre
ORDER BY DESC(?nombreArtistes)
```

Nous pouvons également croiser oeuvres et artistes, par exemple en regardant les couples d'artistes ayant signés le plus d'oeuvres ensemble :

```
SELECT ?nom1 ?nom2 (COUNT(?o) AS ?nb)
WHERE {
  ?o schema:creator ?a1, ?a2 .
  ?a1 foaf:name ?nom1 . ?a2 foaf:name ?nom2 .
  FILTER(STR(?a1) < STR(?a2))
}
GROUP BY ?nom1 ?nom2 ORDER BY DESC(?nb) LIMIT 20
```

Ou en remarquant le nombre d'oeuvres ayant été publiées à titre posthume de l'artiste :

```
SELECT ?nom ?titre ?annee ?mort
WHERE {
  ?o schema:creator ?a ; schema:name ?titre ; schema:dateCreated ?annee .
  ?a foaf:name ?nom ; schema:deathDate ?mort .
  FILTER(xsd:integer(?mort) > 0 && ?annee > xsd:integer(?mort))
}
```
