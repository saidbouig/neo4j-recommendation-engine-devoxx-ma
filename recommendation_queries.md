# Content-Based Filtering ( relative ):
Donner des recommandations sur des noeuds similaires à un noeud en question ( filmes qui sont similaires au film sélectionné )
    
    
    MATCH p=(m:Movie { title: "The Matrix"})-[:HAS_CREDIT|:PRODUCED_BY|:PRODUCED_IN|:SPOKEN_LANGUAGE|:HAS_GENRE*2]-()
    RETURN p LIMIT 100
    

## Similarité basée sur les genres communs

Trouver les films les plus similaires à un film on se basant sur les genres communs:
    
    MATCH (m:Movie)<-[:HAS_GENRE]-(g:Genre)-[:HAS_GENRE]->(rec:Movie)
    WHERE m.title = "The Matrix"
    WITH rec, COLLECT(g.name) AS genres, COUNT(*) AS communGenres
    RETURN rec.title, genres, communGenres
    ORDER BY communGenres DESC LIMIT 10;

## Weighted Content Algorithm

Ajouter du poids sur des relations selon l'importance d'une relation pour calculer la similarité entre les films : 

    Trouver les films qui sont similaires au film The Matrix par les genres communs

    MATCH (m:Movie) WHERE m.title = "The Matrix"
    MATCH (m)-[:HAS_GENRE]-(g:Genre)-[:HAS_GENRE]-(rec:Movie)
    WITH m, rec, COUNT(*) AS gs
    OPTIONAL MATCH (m)-[:HAS_CREDIT]-(c:Credit)-[:CASTED]-(a:Cast)-[:CASTED]-(cc:Credit)-[:HAS_CREDIT]-(rec)
    WITH m, rec, gs, COUNT(a) AS as
    RETURN rec.title AS recommendation, (5*gs)+(3*as) AS score ORDER BY score DESC LIMIT 100

## Jacard index

Quels sont les films qui sont similaires au film The Matrix on se basant sur l'index de Jaccard sur des genres ?
    
    MATCH (m:Movie {title: "The Matrix"})-[:HAS_GENRE]-(g:Genre)-[:HAS_GENRE]-(other:Movie)
    WITH m, other, COUNT(g) AS intersection, COLLECT(g.name) AS i
    MATCH (m)-[:HAS_GENRE]-(mg:Genre)
    WITH m,other, intersection,i, COLLECT(mg.name) AS s1
    MATCH (other)-[:HAS_GENRE]-(og:Genre)
    WITH m,other,intersection,i, s1, COLLECT(og.name) AS s2

    WITH m,other,intersection,s1,s2

    WITH m,other,intersection,s1+filter(x IN s2 WHERE NOT x IN s1) AS union, s1, s2

    RETURN m.title, other.title, s1,s2,((1.0*intersection)/SIZE(union)) AS jaccard ORDER BY jaccard DESC LIMIT 100


On peut appliquer la même méthode sur d'autres features comme le genre, le producteur et le pays de production 

    MATCH (m:Movie {title: "The Matrix"})-[:HAS_GENRE|:PRODUCED_IN|:PRODUCED_BY]-(t)-[:HAS_GENRE|:PRODUCED_IN|:PRODUCED_BY]-(other:Movie)
    WITH m, other, COUNT(t) AS intersection, COLLECT(t.name) AS i
    MATCH (m)-[:IN_GENRE|:PRODUCED_IN|:PRODUCED_BY]-(mt)
    WITH m,other, intersection,i, COLLECT(mt.name) AS s1
    MATCH (other)-[:IN_GENRE|:PRODUCED_IN|:PRODUCED_BY]-(ot)
    WITH m,other,intersection,i, s1, COLLECT(ot.name) AS s2

    WITH m,other,intersection,s1,s2

    WITH m,other,intersection,s1+filter(x IN s2 WHERE NOT x IN s1) AS union, s1, s2

    RETURN m.title, other.title, s1,s2,((1.0*intersection)/SIZE(union)) AS jaccard ORDER BY jaccard DESC LIMIT 100

# User-Based Filtering ( collaborative ):
    
On se basant sur les avis et évaluations du réseau  d'utilisateurs pour trouver des éléments à recommander 


tout les évaluations de l'utilisateur avec id = 1

    MATCH (u:User {id:1})
    MATCH (u)-[r:RATED]->(m:Movie)
    RETURN *;

Calculer la moyenne des évaluations de cet utilisateur

    MATCH (u:User {id: 1})
    MATCH (u)-[r:RATED]->(m:Movie)
    RETURN avg(r.rating) AS average;

Les films que l'utilisateur a évalué avec une note supérieur à la moyenne des ses évaluations     

    MATCH (u:User {id:30})
    MATCH (u)-[r:RATED]->(m:Movie)
    WITH u, avg(r.rating) AS average
    MATCH (u)-[r:RATED]->(m:Movie)
    WHERE r.rating > average
    RETURN *;

Les films que les gens ont regardé et qu'il ont regardé mon film :
    
    MATCH (m:Movie {title: "The Matrix"})<-[:RATED]-(u:User)-[:RATED]->(rec:Movie)
    RETURN rec.title AS recommendation, COUNT(*) AS usersWhoAlsoWatchedThisMovie
    ORDER BY usersWhoAlsoWatchedThisMovie DESC LIMIT 25

## Recommandation personnalisée basée sur les genres

Si on sait les films qu'un utilisateur à regardé, on pourra utiliser cette information pour recommander des films similaires 

    MATCH (u:User {id:1})-[r:RATED]->(m:Movie),
    (m)-[:HAS_GENRE]-(g:Genre)-[:HAS_GENRE]-(rec:Movie)
    WHERE NOT EXISTS( (u)-[:RATED]->(rec) )
    WITH rec, [g.name, COUNT(*)] AS scores
    RETURN rec.title AS recommendation, rec.release_date AS year,
    COLLECT(scores) AS scoreComponents,
    REDUCE (s=0,x in COLLECT(scores) | s+x[1]) AS score
    ORDER BY score DESC LIMIT 10




# Collaborative Filtering The Wisdom of Crowds

## Cosine Similarity
 Les utilisateurs les plus similaires avec Cosine Similarity

    MATCH (p1:User {id:30})-[x:RATED]->(m:Movie)<-[y:RATED]-(p2:User)
    WITH COUNT(m) AS numbermovies, SUM(x.rating * y.rating) AS xyDotProduct,
    SQRT(REDUCE(xDot = 0.0, a IN COLLECT(x.rating) | xDot + a^2)) AS xLength,
    SQRT(REDUCE(yDot = 0.0, b IN COLLECT(y.rating) | yDot + b^2)) AS yLength,
    p1, p2 WHERE numbermovies > 10
    RETURN p1.id, p2.id, xyDotProduct / (xLength * yLength) AS sim
    ORDER BY sim DESC
    LIMIT 100;

## Pearson Similarity

Trouvez les utilisateurs les plus similaires à l'utilisateur avec id=10 en utilisant Pearson: 

    MATCH (u1:User {id:10})-[r:RATED]->(m:Movie)
    WITH u1, avg(r.rating) AS u1_mean

    MATCH (u1)-[r1:RATED]->(m:Movie)<-[r2:RATED]-(u2)
    WITH u1, u1_mean, u2, COLLECT({r1: r1, r2: r2}) AS ratings WHERE size(ratings) > 10

    MATCH (u2)-[r:RATED]->(m:Movie)
    WITH u1, u1_mean, u2, avg(r.rating) AS u2_mean, ratings

    UNWIND ratings AS r

    WITH sum( (r.r1.rating-u1_mean) * (r.r2.rating-u2_mean) ) AS nom,
        sqrt( sum( (r.r1.rating - u1_mean)^2) * sum( (r.r2.rating - u2_mean) ^2)) AS denom,
        u1, u2 WHERE denom <> 0

    RETURN u1.id, u2.id, nom/denom AS pearson
    ORDER BY pearson DESC LIMIT 100

## Simple Collaborative Filtering

Les films évalués par des utilisateurs qui on évalué mes films :

    MATCH (u:User {id:30})-[:RATED]->(:Movie)<-[:RATED]-(o:User)
    MATCH (o)-[:RATED]->(rec:Movie)
    WHERE NOT EXISTS( (u)-[:RATED]->(rec) )
    RETURN rec.title
    LIMIT 25

Les films évalués par des utilisateurs qui on évalué mes films en prenant en considération les genres que j'aime:

    MATCH (u:User {id:30})-[r:RATED]->(m:Movie)
    WITH u, avg(r.rating) AS mean

    MATCH (u)-[r:RATED]->(m:Movie)-[:HAS_GENRE]-(g:Genre)
    WHERE r.rating > mean

    WITH u, g, COUNT(*) AS score

    MATCH (g)-[:HAS_GENRE]-(rec:Movie)
    WHERE NOT EXISTS((u)-[:RATED]->(rec))

    RETURN rec.title AS recommendation, rec.release_date AS year, COLLECT(DISTINCT g.name) AS genres, SUM(score) AS sscore
    ORDER BY sscore DESC LIMIT 10


## kNN k Nearest Neighbors

Qui sont les 10 utilisateurs ayant des goûts dans les films les plus similaires aux miens? Quels films ont-ils hautement apprécié que je n’ai pas encore vus?


    MATCH (u1:User {id:30})-[r:RATED]->(m:Movie)
    WITH u1, avg(r.rating) AS u1_mean

    MATCH (u1)-[r1:RATED]->(m:Movie)<-[r2:RATED]-(u2)
    WITH u1, u1_mean, u2, COLLECT({r1: r1, r2: r2}) AS ratings WHERE size(ratings) > 10

    MATCH (u2)-[r:RATED]->(m:Movie)
    WITH u1, u1_mean, u2, avg(r.rating) AS u2_mean, ratings

    UNWIND ratings AS r

    WITH sum( (r.r1.rating-u1_mean) * (r.r2.rating-u2_mean) ) AS nom,
        sqrt( sum( (r.r1.rating - u1_mean)^2) * sum( (r.r2.rating - u2_mean) ^2)) AS denom,
        u1, u2 WHERE denom <> 0

    WITH u1, u2, nom/denom AS pearson
    ORDER BY pearson DESC LIMIT 10
    
    MATCH (u2)-[r:RATED]->(m:Movie) WHERE NOT EXISTS( (u1)-[:RATED]->(m) )

    RETURN m.title, SUM( pearson * r.rating) AS score
    ORDER BY score DESC LIMIT 25
