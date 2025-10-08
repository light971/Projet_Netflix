# Netflix Movies and TV Shows Data Analysis using SQL

![](https://github.com/light971/netflix_sql_project/blob/main/logo.png)

## Overview
This project involves a comprehensive analysis of Netflix's movies and TV shows data using SQL. The goal is to extract valuable insights and answer various business questions based on the dataset. The following README provides a detailed account of the project's objectives, business problems, solutions, findings, and conclusions.

## Objectives

- Analyze the distribution of content types (movies vs TV shows).
- Identify the most common ratings for movies and TV shows.
- List and analyze content based on release years, countries, and durations.
- Explore and categorize content based on specific criteria and keywords.

## Dataset

The data for this project is sourced from the Kaggle dataset:

- **Dataset Link:** [Movies Dataset](https://www.kaggle.com/datasets/shivamb/netflix-shows?resource=download)

## Schema

```sql
DROP TABLE IF EXISTS netflix;
CREATE TABLE netflix
(
    show_id      VARCHAR(5),
    type         VARCHAR(10),
    title        VARCHAR(250),
    director     VARCHAR(550),
    casts        VARCHAR(1050),
    country      VARCHAR(550),
    date_added   VARCHAR(55),
    release_year INT,
    rating       VARCHAR(15),
    duration     VARCHAR(15),
    listed_in    VARCHAR(250),
    description  VARCHAR(550)
);
```

## Business Problems and Solutions

### 1. Count the Number of Movies vs TV Shows

```sql
SELECT 
    type,
    COUNT(*)
FROM netflix
GROUP BY 1;
```

**Objective:** Determine the distribution of content types on Netflix.

### 2. Find the Most Common Rating for Movies and TV Shows

```sql
WITH RatingCounts AS (
    SELECT 
        type,
        rating,
        COUNT(*) AS rating_count
    FROM netflix
    GROUP BY type, rating
),
RankedRatings AS (
    SELECT 
        type,
        rating,
        rating_count,
        RANK() OVER (PARTITION BY type ORDER BY rating_count DESC) AS rank
    FROM RatingCounts
)
SELECT 
    type,
    rating AS most_frequent_rating
FROM RankedRatings
WHERE rank = 1;
```

**Objective:** Identify the most frequently occurring rating for each type of content.

### 3. List All Movies Released in a Specific Year (e.g., 2020)

```sql
SELECT * 
FROM netflix
WHERE release_year = 2020;
```

**Objective:** Retrieve all movies released in a specific year.

### 4. Find the Top 5 Countries with the Most Content on Netflix

```sql
WITH RECURSIVE
  SplitCountry(title, rest) AS (
    -- Initialiser la récursion en ajoutant une virgule à la fin pour traiter le dernier élément
    SELECT
      title,
      country || ','
    FROM
      netflix
    UNION ALL
    -- Étape récursive : diviser la chaîne
    SELECT
      title,
      SUBSTR(rest, INSTR(rest, ',') + 1)  -- Reste de la chaîne
    FROM
      SplitCountry
    WHERE
      INSTR(rest, ',') > 0 -- Continuer tant qu'il y a des virgules à traiter
  ),
  UnnestedCountries AS (
    -- Extraire chaque pays de la colonne 'rest' qui a été réduite à une seule valeur
    SELECT
      TRIM(SUBSTR(rest, 1, INSTR(rest, ',') - 1)) AS country_name
    FROM
      SplitCountry
    WHERE
      INSTR(rest, ',') > 0 -- Sélectionner uniquement les parties valides avant le délimiteur
  )
-- Requête Finale (Simule le GROUP BY et le TOP 5 de votre requête originale)
SELECT
  country_name,
  COUNT(*) AS total_content
FROM
  UnnestedCountries
WHERE
  country_name IS NOT NULL AND country_name != '' -- Filtrer les valeurs NULL ou vides
GROUP BY
  country_name
ORDER BY
  total_content DESC
  LIMIT 5;
```

**Objective:** Identify the top 5 countries with the highest number of content items.

### 5. Identify the Longest Movie

```sql
SELECT type, duration
FROM netflix
WHERE type = 'Movie' 
        AND 
        duration = (SELECT MAX(duration) FROM netflix)
GROUP BY 1;
```

**Objective:** Find the movie with the longest duration.

### 6. Find Content Added in the Last 5 Years

```sql
SELECT *
FROM netflix
WHERE date_added >= DATE('now', '-5 year');
```

**Objective:** Retrieve content added to Netflix in the last 5 years.

### 7. Find All Movies/TV Shows by Director 'Rajiv Chilaka'

```sql
WITH RECURSIVE
  SplitDirectors(show_id, director_name, rest) AS (
    -- Initialisation : Prépare la chaîne pour le split
    SELECT
      show_id,
      NULL, -- Placeholders pour le nom extrait (non utilisé ici)
      TRIM(director) || ',' -- Assure une virgule et un nettoyage de base
    FROM
      netflix
    UNION ALL
    -- Récursion : Extrait le premier nom et passe le reste
    SELECT
      show_id,
      TRIM(SUBSTR(rest, 1, INSTR(rest, ',') - 1)), -- Nom extrait
      SUBSTR(rest, INSTR(rest, ',') + 1)  -- Reste de la chaîne
    FROM
      SplitDirectors
    WHERE
      INSTR(rest, ',') > 0
  )
SELECT
  T1.* -- Sélectionne toutes les colonnes de la table netflix
FROM
  netflix AS T1
JOIN
  SplitDirectors AS T2 ON T1.show_id = T2.show_id
WHERE
  T2.director_name = 'Rajiv Chilaka'; -- Filtre sur le nom dénormalisé
```

**Objective:** List all content directed by 'Rajiv Chilaka'.

### 8. List All TV Shows with More Than 5 Seasons

```sql
SELECT
  *
FROM
  netflix
WHERE
  type = 'TV Show'
  -- Extrait le nombre de saisons de la chaîne (ex: '7 Seasons')
  AND CAST(
      SUBSTR(
          duration,
          1,
          INSTR(duration, ' ') - 1
      )
      AS INTEGER
  ) > 5;

```

**Objective:** Identify TV shows with more than 5 seasons.

### 9. Count the Number of Content Items in Each Genre

```sql
WITH RECURSIVE
  SplitGenres(ID, rest) AS (
    -- Étape 1 : Initialisation
    SELECT
      show_id, -- Utilisez un identifiant unique de la table
      listed_in || ',' -- Ajouter un délimiteur final pour simplifier le traitement du dernier élément
    FROM
      netflix
    UNION ALL
    -- Étape 2 : Récursion
    SELECT
      ID,
      SUBSTR(rest, INSTR(rest, ',') + 1)  -- Reste de la chaîne après le délimiteur
    FROM
      SplitGenres
    WHERE
      INSTR(rest, ',') > 0 -- Condition d'arrêt : continuer tant qu'il y a des virgules
  ),
  UnnestedGenres AS (
    -- Étape 3 : Extraction des éléments individuels
    SELECT
      TRIM(
        SUBSTR(rest, 1, INSTR(rest, ',') - 1)
      ) AS genre_name
    FROM
      SplitGenres
    WHERE
      INSTR(rest, ',') > 0 -- Ne pas inclure les lignes vides
  )
-- Étape 4 : Agrégation (équivalent au GROUP BY de la requête originale)
SELECT
  genre_name AS genre,
  COUNT(*) AS total_content
FROM
  UnnestedGenres
WHERE
  genre_name IS NOT NULL AND genre_name != '' -- Nettoyage final
GROUP BY
  genre_name
ORDER BY
  total_content DESC;
```

**Objective:** Count the number of content items in each genre.

### 10.Find each year and the average numbers of content release in India on netflix. 
return top 5 year with highest avg content release!

```sql
SELECT 
    country,
    release_year,
    COUNT(show_id) AS total_release,
    ROUND(
        COUNT(show_id)::numeric /
        (SELECT COUNT(show_id) FROM netflix WHERE country = 'India')::numeric * 100, 2
    ) AS avg_release
FROM netflix
WHERE country = 'India'
GROUP BY country, release_year
ORDER BY avg_release DESC
LIMIT 5;
```

**Objective:** Calculate and rank years by the average number of content releases by India.

### 11. List All Movies that are Documentaries

```sql
SELECT * 
FROM netflix
WHERE listed_in LIKE '%Documentaries';
```

**Objective:** Retrieve all movies classified as documentaries.

### 12. Find All Content Without a Director

```sql
SELECT * 
FROM netflix
WHERE director IS NULL;
```

**Objective:** List content that does not have a director.

### 13. Find How Many Movies Actor 'Salman Khan' Appeared in the Last 10 Years

```sql
SELECT
    *
FROM
    netflix
WHERE
    -- 1. Search for 'Salman Khan' (LIKE is case-insensitive in SQLite by default for ASCII)
    casts LIKE '%Salman Khan%'
    AND
    -- 2. Compare release_year with the current year minus 10
    release_year > (CAST(STRFTIME('%Y', 'now') AS INTEGER) - 10);
    
```

**Objective:** Count the number of movies featuring 'Salman Khan' in the last 10 years.

### 14. Find the Top 10 Actors Who Have Appeared in the Highest Number of Movies Produced in India

```sql
WITH RECURSIVE SplitCasts(show_id, actor, rest) AS (
  -- Étape 1 : initialisation
  SELECT
    show_id,
    TRIM(SUBSTR(casts, 1, INSTR(casts || ',', ',') - 1)) AS actor,
    SUBSTR(casts || ',', INSTR(casts || ',', ',') + 1) AS rest
  FROM
    netflix
  WHERE
    country = 'India'

  UNION ALL

  -- Étape 2 : récursion — découpe les acteurs suivants
  SELECT
    show_id,
    TRIM(SUBSTR(rest, 1, INSTR(rest, ',') - 1)) AS actor,
    SUBSTR(rest, INSTR(rest, ',') + 1)
  FROM
    SplitCasts
  WHERE
    rest <> ''
    AND INSTR(rest, ',') > 0
)
SELECT
  actor,
  COUNT(*) AS appearances
FROM
  SplitCasts
WHERE
  actor <> ''
GROUP BY
  actor
ORDER BY
  appearances DESC
LIMIT 10;
```

**Objective:** Identify the top 10 actors with the most appearances in Indian-produced movies.

### 15. Categorize Content Based on the Presence of 'Kill' and 'Violence' Keywords

```sql
SELECT 
    category,
    COUNT(*) AS content_count
FROM (
    SELECT 
        CASE 
            WHEN description LIKE '%kill%' OR description LIKE '%violence%' THEN 'Bad'
            ELSE 'Good'
        END AS category
    FROM netflix
) AS categorized_content
GROUP BY category;

```

**Objective:** Categorize content as 'Bad' if it contains 'kill' or 'violence' and 'Good' otherwise. Count the number of items in each category.

## Findings and Conclusion

- **Content Distribution:** The dataset contains a diverse range of movies and TV shows with varying ratings and genres.
- **Common Ratings:** Insights into the most common ratings provide an understanding of the content's target audience.
- **Geographical Insights:** The top countries and the average content releases by India highlight regional content distribution.
- **Content Categorization:** Categorizing content based on specific keywords helps in understanding the nature of content available on Netflix.

This analysis provides a comprehensive view of Netflix's content and can help inform content strategy and decision-making.
