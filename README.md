# Big Query Processor.

ANALYZING THE DBLP DATASET USING NEO4J

Neo4j is the Graph Database which is mainly used for processing the graph storage data and process them in order to build the meaningful insights. The structure of the data in the database is generally in the form of Graph where each node label represents the entity which the edge between them as the relationships between the nodes. These nodes labels and relationship between then can have the properties which define those entities. These properties make the Neo4j stand above all other database management systems. 
	The queries which are used in all other databases are called as Cyphers in Neo4j. With these Cypher all the CRUD operations can be performed on the database nodes. Initially with indexing we can create the node labels of the database and specify the properties so that the data inserted into the database will follow these properties. Neo4j databases support all formats of data such as JSON, CSV, XML etc. so that any form of the data can be imported into the Graph database and can perform the CRUD operations. Neo4j can be used as the local native version which comes as Neo4j Desktop application and also on-demand online version which comes in Neo4j Sandbox. 



STEP-1:- CO-AUTHOR RELATIONSHIP
	As part of the initial step-1 we were asked to create the CO-AUTHORSHIP graph, where there should be a relation CO-AUTHOR between two authors in the Author node labels, when they have collaborated for publishing same Articles and even also while publishing multiple Articles. For that we have executed the following apoc.periodic.iterate query while creating the CO-AUTHOR relationship while setting the property collaborations for that CO-AUTHOR relation.

	CALL apoc.periodic.iterate(
  		"MATCH (a1)<-[:AUTHOR]-(paper)-[:AUTHOR]->(a2:Author)
  		 WITH a1, a2, paper
 ORDER BY a1, paper.year
 RETURN a1, a2, collect(paper)[0].year AS year, count(*) AS collaborations",
 "MERGE (a1)-[coauthor:CO_AUTHOR {year: year}]-(a2)
 SET co-author.collaborations = collaborations",
 {batchSize: 100}
         );

The CO-AUTHOR relationship graph with the property “collaborations” and which “year” they collaborated is created as shown 
![image](https://github.com/user-attachments/assets/3ab01630-3ecd-4e0f-81d0-b1bbcc5037b8)


STEP-2 :- KEYWORD DISCOVERY
	As part of Step-2 given to us, we were asked to list all the Authors who are working in a particular research area, when the user enters the research topic as the user input value. Also, we need to find the score and relevance of that particular Author.
Score:- Score of the Author estimates the weight of that keyword among all other author’s publication records. We have calculated score of the particular author as follows :-

No. of Articles in which the respective Author is involved in publication in given research topic
     Total no. of Articles which the respective Author has published in all the research topics

Relevance:- Relevance estimates the prolificacy of the author within the whole DBLP community that has been working on that topic. We have calculated the relevance value as follows:-
        Total no. of Articles has published by the respective Author in the given research topic.
Total No. of Articles published by all other Authors in the given research topic.
From the above formula, it is clearly observed that this SCORE and RELEVANCE are not constant for the particular Author. They are the values of the Author’s research work to a particular research topic. So, it is required that we need to calculate the SCORE and RELEVANCE with respect to the Author’s contribution to the particular research topic.
	In order to calculate the score and relevance of the Author’s working to a particular research topic we have again created a new relationship named as RESEARCH_TOPIC, where the Author working on a particular keyword has published the Article in that research area. The following cypher is what we used to create the relationship and calculate the SCORE of the Author’s contribution to the particular RESEARCH_TOPIC.
	MATCH (a:Author) <- [:AUTHOR] - (p:Article) - [r:CONTAIN] -> (k:keyword)
WITH a,collect(k) as keycoll, count(k.key) as keynum
UNWIND keycoll as k
WITH a,k,(100*count(k))/keynum as percentage
CREATE (a) - [:RESEARCH_TOPIC {score : percentage}] -> (k);
 
The following is the cypher which we have written in order to calculate the relevance based on the formula stated above.
MATCH (p:Article)-[r:CONTAIN] -> (k:keyword)
WITH k,count(r) as num_pub
MATCH m=(a:Author) <- [:AUTHOR] - (p:Article) - [l:CONTAIN] -> (k)
WITH a,k, (count(m)*100)/(num_pub*1.0) as relevance
MATCH (k) <- [r: RESEARCH_TOPIC] - (a)
SET r.relevence = relevance

After successful execution of the above the relationship RESEARCH_TOPIC has been created with node properties as score and relevance which means the respective Author contribution to the particular research area which they worked in his whole career as shown
	
					         
As part of STEP-1, we need to display all the Authors who are working the given user input’s research area along with his score and relevance. The following is the cypher which we used to get the desired output for the research topic “database”
	MATCH (k:keyword) <- [s:RESEARCH_TOPIC] - (a:Author)
WHERE k.key = "database"
WITH k,a,s 
      RETURN k.key,a.name,s.score,s.relevence ORDER BY s.relevence asc, s.score asc limit 5 

After successful execution of the Cypher
					 
STEP-3:- RESEARCHER PROFILING
	As part of this step, the user inputs the Author name of their desire and we need to find out all the Authors who have been working in that respective research area in which that Author is working. We need to order the collaborator Authors as per their score and relevance. The Cypher which we have written in order to get the desired output is as follows:
MATCH (b:Author) - [l:RESEARCH_TOPIC] -> (k:keyword) <- [r:RESEARCH_TOPIC] - (a:Author)
WHERE a.name='Ling Chen'
WITH b,k,l.score as sugg_author_score,
    l.relevence as sugg_author_relevence,
    r.score as author_score,
   r.relevence as author_relevence
WITH collect([b,author_relevence,author_score,sugg_author_relevence,
        sugg_author_score]) as researchers_data, k
UNWIND researchers_data[0..3] AS r
WITH k,(r[0]).name as name, r[3] as sugg_author_relevence, r[4] as
    sugg_author_score, r[1] as author_relevence, r[2] as author_score
RETURN k.key as Topic, name, author_score, author_relevence,
    sugg_author_relevence, sugg_author_score

This Cypher first matches the Author’s research area which he is working on, and then finds out all other author who are working in that research topic.

					
STEP-4:- INFLUENCING AUTHOR

	In this step, the user enters the Research topic of this interest, the tool should extract all the authors working in that research topic with the respective Page Rank value. The higher the page rank it means the more influential is the Author. For calculating the Page rank of the respective Author we have used the in-built page rank algorithm present in the Graph Data science Library of the Neo4j. In order to use this algorithm we need to first install the Neo4j Graph Data Science Library Plugin which can be done from the Desktop Home Screen Application as shown 



After successfully installation, we need to following series of Cypher in order to set the Page Rank property to the corresponding Authors.
CALL gds.graph.create
(
  'myGraph',
  'Author',
  'CO_AUTHOR',
  {
    relationshipProperties: 'collaborations'
  }
)

We are creating a Virtual graph called myGraph, taking the input from the Author Node Labels and the CO_AUTHOR relationship with the property of how many collaborations they have made between them. 
CALL gds.pageRank.write.estimate('myGraph', {
  writeProperty: 'pageRank',
  maxIterations: 20,
  dampingFactor: 0.85
})

Now, gds.pageRank is the in-built method which calculates the pagerank of the particular author in the node label. We are calculating the page rank by iterating over all the links in which the Author is involved of maxiterations set to 20 and the damping Faction as 0.85.

CALL gds.pageRank.stream('myGraph')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS name, score
ORDER BY score DESC, name ASC

With the above cypher the pagerank property is calculated for all the node labels in the graph. In order to write the value of the Pagerank to the corresponding Author we have used the below cypher which uses the in-built pagerank.write() method to write the property to the node labels.

		CALL gds.pageRank.write('myGraph', {
  maxIterations: 20,
  dampingFactor: 0.85,
  writeProperty: 'pagerank'
})

By this the pagerank property has been set to all the Authors in the Graph. In order to get the influential Authors in a particular Research Topic based on the order of Page rank we have ran the following Cypher in order to get the desired output.
	
	MATCH (k:keyword) <- [s:RESEARCH_TOPIC] - (a:Author) WHERE k.key ="database" WITH k,a,s RETURN a.name,a.pagerank ORDER BY a.pagerank DESC

On successful execution of the above Cypher, the output for the Research Topic “database” is as shown in the. The order is in Descending order, which means the Author at the top is the most influential person in that particular research topic.
