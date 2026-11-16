# Projet Dana | Groupe CAZ, OULAI, ATTEYEH ABI

## **1. Première partie**

### 1.0 Installation des outils

Installation du serveur Fuseki, et de tarql pour convertir le fichier CSV en RDF.

### 1.1 Conversion JSON en CSV


Conversion du fichier artists.json (de base) en un fichier CSV exploitable par tarsQL à l'aide du PowerShell Windows :


```PowerShell
Get-Content -Path "artists.json" -Encoding UTF8 | ConvertFrom-Json | ForEach-Object { $_ } | Export-Csv -Path "artistsRaw.csv" -NoTypeInformation -Encoding UTF8 
```

### 1.2 Nettoyage du .CSV à l'aide de Python (Pandas)

Une fois le fichier converti, on procède à un nettoyage à l'aide d'un script python et de la librarie Pandas : 
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
### 2.1 Requêtes "intéressantes"

La première requêtes intéressante consiste à voir la répartion des nationalités au sein du jeu de données :

```
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX schema: <https://schema.org/>

SELECT ?nationalite (COUNT(DISTINCT ?artiste) AS ?nombreArtistes)
WHERE {
    ?artiste foaf:name ?nom;
    schema:nationality ?nationalite.
}
GROUP BY ?nationalite
ORDER BY DESC(?nombreArtistes)
LIMIT 10
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
