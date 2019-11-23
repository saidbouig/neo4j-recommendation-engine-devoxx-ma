# Construire votre propre moteur de recommandation avec Neo4j

## Clone project

## Clone project

    git clone https://github.com/saidbouig/neo4j_recommendation_engine_devoxx

## Setup Python environment / install packages

you need Python and pip installed 

    virtualenv env
    source ./env/bin/activate
    pip install -r requirements.txt

## Run Neo4j container 

you need docker and docker-compose installed 

    docker-compose up

## Get TMDB <https://www.themoviedb.org/> API key and replace it in constants.py

    KEY='YOUR_API_KEY'

## Run movie_ingestor.py to get movies from tmdb

    python movie_ingestor.py

## Get User ratings from Movielens Dataset

Goto 
    
    localhost:7474/browser/

Then copy this query to import ratings

    LOAD CSV FROM 'https://gist.githubusercontent.com/saidbouig/9bbe7b22730fe93f19ca254df93de503/raw/3bdd87939401151f7a1aab133f83bcbf542f5782/user_ratings.edges' AS row

    WITH toInteger(row[0]) AS userId, toInteger(row[1]) AS movieId, toFloat(row[2]) AS rating, toInteger(row[3]) AS timeStamp

    MATCH(m: Movie {id: movieId})
    MERGE(u: User {id: userId})

    MERGE(u)-[rel:RATED {rating: rating, timeStamp: timeStamp}] -> (m)
    RETURN count(rel)


## Useful links 


    Cypher langauge documentaion

    https://neo4j.com/docs/cypher-manual/current/

    TMDB API KEY

    https://www.themoviedb.org/settings/api

    TMDB API documentation

    https://developers.themoviedb.org/3/getting-started/introduction


