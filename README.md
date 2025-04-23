# Analyzing the DBLP Dataset Using Neo4j

This project analyzes the DBLP dataset using Neo4j, a graph database optimized for processing graph-structured data to derive meaningful insights. The dataset is modeled as nodes (representing entities like articles, authors, and keywords) and relationships (representing connections like authorship or citations). The project includes a web application built with Flask, HTML, CSS, JavaScript, and Bootstrap to provide a user-friendly interface for querying and visualizing the data.


## Neo4j Overview
Neo4j stores data as a graph, with nodes and relationships that can have properties. It uses Cypher queries for CRUD operations and supports data imports in formats like JSON, CSV, and XML. Neo4j is available as a local Desktop application or an online Sandbox. This project uses the Community Edition of Neo4j Desktop.

## Setup and Installation
1. **Install Neo4j Desktop**:
   - Download the Community Edition from the [official Neo4j website](https://neo4j.com/download/).
   - Run the executable and enter the activation key provided during registration.
   - Start Neo4j Desktop to access the project dashboard.

2. **Configure APOC Plugin**:
   - Navigate to the Plugins tab in Neo4j Desktop and install the APOC plugin.
   - Restart the database to apply changes.
   - Enable file imports by setting `apoc.import.file.enabled=true` in the `neo4j.conf` file.

3. **Data Import**:
   - Place the DBLP dataset (`dblp-ref-3.json`) in the import folder: `/users/[username]/Neo4jDesktop/relate-data/dbmss/[DB-NAME]/import/`.
   - Create indexes on `Article` (property: `index`) and `Author` (property: `name`) to optimize import performance.
   - Run the following Cypher query to import data and create nodes (`Article`, `Author`, `Venue`) and relationships (`AUTHOR`, `VENUE`, `CITED`):

   ```cypher
   CALL apoc.periodic.iterate(
       'UNWIND ["dblp-ref-3.json"] AS file
        CALL apoc.load.json("./" + file)
        YIELD value
        RETURN value',
       'MERGE (a:Article {index:value.id})
        SET a += apoc.map.clean(value, ["id","authors","references","venue"], [0])
        WITH a, value.authors as authors, value.references AS citations, value.venue AS venue
        MERGE (v:Venue {name: venue})
        MERGE (a)-[:VENUE]->(v)
        FOREACH(author in authors |
          MERGE (b:Author {name:author})
          MERGE (a)-[:AUTHOR]->(b))
        FOREACH(citation in citations |
          MERGE (cited:Article {index:citation})
          MERGE (a)-[:CITED]->(cited))',
       {batchSize: 1000, iterateList: true}
   );
   ```
![image](https://github.com/user-attachments/assets/2adf4122-d644-45c6-9333-95dd8cc59ad6)

## Project Steps

### Step 1: Co-Author Relationship
- Created a `CO_AUTHOR` relationship between authors who collaborated on articles, with properties `year` and `collaborations`.
- Cypher query:
  ```cypher
  CALL apoc.periodic.iterate(
      'MATCH (a1)<-[:AUTHOR]-(paper)-[:AUTHOR]->(a2:Author)
       WITH a1, a2, paper
       ORDER BY a1, paper.year
       RETURN a1, a2, collect(paper)[0].year AS year, count(*) AS collaborations',
      'MERGE (a1)-[coauthor:CO_AUTHOR {year: year}]-(a2)
       SET coauthor.collaborations = collaborations',
      {batchSize: 100}
  );
  ```
- Imported keywords from a CSV file to identify research topics and created a `CONTAIN` relationship between articles and keywords.
- ![image](https://github.com/user-attachments/assets/ef168101-297c-4ea5-b3f1-4d6869046028)


### Step 2: Keyword Discovery
- Lists authors working in a user-specified research topic, with their `score` and `relevance`.
- **Score**: Proportion of an author's articles in the given topic relative to their total publications.
- **Relevance**: Proportion of an author's articles in the topic relative to all articles in that topic.
- Cypher queries:
  ```cypher
  MATCH (a:Author) <- [:AUTHOR] - (p:Article) - [r:CONTAIN] -> (k:keyword)
  WITH a, collect(k) as keycoll, count(k.key) as keynum
  UNWIND keycoll as k
  WITH a, k, (100 * count(k)) / keynum as percentage
  CREATE (a) - [:RESEARCH_TOPIC {score: percentage}] -> (k);

  MATCH (p:Article) - [r:CONTAIN] -> (k:keyword)
  WITH k, count(r) as num_pub
  MATCH m = (a:Author) <- [:AUTHOR] - (p:Article) - [l:CONTAIN] -> (k)
  WITH a, k, (count(m) * 100) / (num_pub * 1.0) as relevance
  MATCH (k) <- [r:RESEARCH_TOPIC] - (a)
  SET r.relevance = relevance;

  MATCH (k:keyword) <- [s:RESEARCH_TOPIC] - (a:Author)
  WHERE k.key = "database"
  WITH k, a, s
  RETURN k.key, a.name, s.score, s.relevance ORDER BY s.relevance ASC, s.score ASC LIMIT 5;
  ```
  ![image](https://github.com/user-attachments/assets/48926a10-4670-4ec0-8e03-981f98c93bab)


### Step 3: Researcher Profiling
- Identifies authors collaborating in the same research area as a user-specified author, ordered by score and relevance.
- Cypher query:
  ```cypher
  MATCH (b:Author) - [l:RESEARCH_TOPIC] -> (k:keyword) <- [r:RESEARCH_TOPIC] - (a:Author)
  WHERE a.name = 'Ling Chen'
  WITH b, k, l.score as sugg_author_score, l.relevance as sugg_author_relevance,
       r.score as author_score, r.relevance as author_relevance
  WITH collect([b, author_relevance, author_score, sugg_author_relevance, sugg_author_score]) as researchers_data, k
  UNWIND researchers_data[0..3] AS r
  WITH k, (r[0]).name as name, r[3] as sugg_author_relevance, r[4] as sugg_author_score,
       r[1] as author_relevance, r[2] as author_score
  RETURN k.key as Topic, name, author_score, author_relevance, sugg_author_relevance, sugg_author_score;
  ```

### Step 4: Influential Authors
- Identifies influential authors in a user-specified research topic using PageRank from the Neo4j Graph Data Science Library.
- Install the Graph Data Science Library plugin and run:
  ```cypher
  CALL gds.graph.create(
      'myGraph',
      'Author',
      'CO_AUTHOR',
      { relationshipProperties: 'collaborations' }
  );

  CALL gds.pageRank.write('myGraph', {
      maxIterations: 20,
      dampingFactor: 0.85,
      writeProperty: 'pagerank'
  });

  MATCH (k:keyword) <- [s:RESEARCH_TOPIC] - (a:Author)
  WHERE k.key = "database"
  WITH k, a, s
  RETURN a.name, a.pagerank ORDER BY a.pagerank DESC;
  ```

## Web Application
- Built using Flask, HTML, CSS, JavaScript, and Bootstrap.
- Connects to Neo4j via the `neo4j` Python package on port 7687.
- Features:
  - **Keyword Discovery**: Users input a research topic to view authors with their scores and relevance.
  - **Researcher Profiling**: Users input an author’s name to view collaborators in the same research area.
  - **Influential Authors**: Users input a research topic to view top authors ranked by PageRank.
- Flask routes handle form inputs via POST requests, execute Cypher queries, and return results in JSON format for display.

- ![image](https://github.com/user-attachments/assets/d6a5d92e-4d26-4655-baf5-cd3f2121a88c)


## Challenges
1. **APOC Configuration**: Required enabling `apoc.import.file.enabled` to import JSON data.
2. **Data Import Speed**: Indexing node properties reduced import time from 6-8 hours to under a minute.
3. **Researcher Profiling**: Handled incomplete query results by storing data in an array and unwinding it.
4. **Framework Choice**: Used Flask over Django for its API support, suitable for POST-based data fetching.
5. **Data Format**: Converted query results to JSON by storing them in a dictionary for front-end display.

## Conclusion
Graph databases like Neo4j excel in handling data with complex relationships, enabling fast querying and updates on large datasets. They support multiple data formats and are ideal for applications like social network analysis and graph mining. This project demonstrates Neo4j’s capabilities in analyzing the DBLP dataset and delivering insights through an intuitive web interface.
