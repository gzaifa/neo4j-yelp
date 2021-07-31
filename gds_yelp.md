# Neo4j's Graph Data Science (gds) library  
Neo4j's gds library contains graph algorithms which are divided into categories which represent different problem classes.  
| Category      | Description | Algorithms    |
| :---        |    :----   | :---- |
| Centrality      | Are used to determine the importance of distinct nodes in a network.       | Page Rank, Article Rank, Eigenvector Centrality, Betweeness Centrality, Degree Centrality   |
| Community Detection   | Are used to evaluate how groups of nodes are clustered or partitioned, as well as their tendency to strengthen or break apart.         | Louvain, Label Propagation, Weakly Connected Components, Triangle Count, Local Clustering Coefficient      |
| Path finding       | Are used to find the shortest path between two or more nodes or evaluate the availability and quality of paths. | Dijkstra Source-Target, Dijkstra Single-Source, A*, Yen’s algorithm |
| Similarity | Are used to compute the similarity of pairs of different vector-based metrics.  | Node similarity,K-Nearest Neighbors  |

Ensure that the Graph Data Science Library is installed. It can be found under *Plugins* (which is activated by clicking on the database name, e.g. "Movie DBMS..." in my case).
![image](https://user-images.githubusercontent.com/830693/127670097-0c59aca4-7d29-493e-b198-daa52caa0d68.png)  

Some scenarios which I want to run with gds:
<ol>
<li>Find communities within the graph</li>
<li>For these communities (or one of the communities), find the top 10 important nodes(users)</li>
<li>Find in-between nodes(users) which acts as the "bridge" between different communities</li>
</ol>

## Find communities within the graph  
What are the properties that we want in a community:  
* Many edges connecting vertices within a community  
* More connections internally then externally
 
As there are 2,189,457 users and 17,971,548 "FRIENDS" relationships in the graph, we might want to drill down to smaller graphs before running some of the more compute intensive algorithms (for e.g. Triangle with O(n * m) time, where n is the number of nodes and m the number of relationships in the graph).  
We can use the [weakly connected components](https://neo4j.com/docs/graph-data-science/current/algorithms/wcc/) algorithm to separate out the "islands", as it finds sets of nodes where the same set forms a connected component. Read about connected, weakly connected, and strongly connected [here](https://en.wikipedia.org/wiki/Connectivity_(graph_theory)).
<pre>CALL gds.wcc.write({
    nodeProjection: 'User',
    relationshipProjection: {
        FRIENDS_OF: {
            type: 'FRIENDS',
            orientation: 'UNDIRECTED'
        }
    },
    writeProperty: 'component_id'
})</pre>
In the procedure above, the property "component_id" is written to "Users" nodes. It completed relatively fast in 10061ms, finding 1,163,895 components.  
Query the top 10 sets of components using:
<pre>MATCH (u:User) 
RETURN u.component_id, count(*) AS num 
ORDER BY num DESC LIMIT 10</pre>

| u.component_id      | num |
| :---        |    ----:   |
| 109 | 1018074 |
| 112624 | 6 |
| 2003 |6      |
| 318942 |6      |
| 18630           |5      |
| 274299          |5      |
| 293805          |5      |
| 118919          |5      |
| 153377          |5      |
| 108051          |5      |
  
There is a set with 1,018,074 users (around half the size of the original graph) with very many small sets (6 nodes and below). We will concentrate on this set with component_id 109. Create a named graph for use in subsequent query/analysis using:
<pre>CALL gds.graph.create.cypher(
    'com109Graph',
    'MATCH (u:User) WHERE u.component_id = 109 RETURN id(u) AS id',
    'MATCH (u1:User)-[:FRIENDS]-(u2:User) WHERE u1.component_id = 109 AND u2.component_id = 109 RETURN id(u1) AS source, id(u2) AS target'
)</pre>

### Louvain's algorithm
Find the communities within the component using [Louvain's algorithm](https://neo4j.com/docs/graph-data-science/current/algorithms/louvain/). Some things to note:  
* Louvain is a 2 phase algorithm, repeated iteratively and uses modularity (criterion to measure the density of links in a network).
* It is relatively fast at O(n * log n) if n is the number of nodes in the network.
* Works on undirected nodes.
* There are differences between parallel vs non-paralleled processing (param: concurrency which is defaulted to 4). Consistent values are gotten with non-parallel processing (i.e., 1 thread), with parallel processing, values will differ slightly from one run to the next.
* maxIterations (default 10): in two-phase round, the first phase keeps repeating iteratively until either the improvement of Modularity is negligible (tolerance below) or the iterations number is reached as specified here.
* tolerance (default 0.0001): see above. In addition, the smaller the tolerance is, the higher Modularity can be squeezed out. (Modularity: the higher, the better; but more iterations may be needed).  

Let's investigate the effects of parallel/non-parallel processing and varying tolerance level on the output.
First varying the tolerance level:
<pre>UNWIND [0.01, 0.001, 0.0001, 0.00001, 0.000001, 0.0000001, 0.00000001, 0.000000001] as toleranceLevel
CALL gds.louvain.write(
    'com109Graph',
    {
        tolerance: toleranceLevel,
        writeProperty: 'communityId'
    }
)
YIELD ranLevels, computeMillis, communityCount, modularity
RETURN toleranceLevel, computeMillis , ranLevels, communityCount, modularity</pre>  
First run:
|"toleranceLevel"|"computeMillis"|"ranLevels"|"communityCount"|"modularity"      |
|:---|---:|---:|---:|---:|
|0.01            |13392          |2          |2101            |0.6537906376520618|
|0.001           |17595          |3          |226             |0.6540843411429211|
|0.0001          |18588          |3          |225             |0.6554073088692922|
|0.00001         |15854          |3          |180             |0.6543239969229836|
|0.000001        |15588          |4          |176             |0.6543005860622277|
|0.0000001       |15763          |4          |194             |0.6552706984550967|
|0.00000001      |16584          |4          |180             |0.6562799279592473|
|0.000000001     |15485          |5          |213             |0.6456996115708254|
  
Second run:
|"toleranceLevel"|"computeMillis"|"ranLevels"|"communityCount"|"modularity"      |
|:---|---:|---:|---:|---:|
|0.01            |15048          |2          |2101            |0.6546005425114918|
|0.001           |16178          |2          |1872            |0.6548609101678122|
|0.0001          |17140          |3          |162             |0.6548731461025318|
|0.00001         |16773          |3          |235             |0.6553389890732767|
|0.000001        |15700          |4          |144             |0.6545002487813525|
|0.0000001       |15452          |4          |182             |0.6557713727646982|
|0.00000001      |16448          |4          |180             |0.6513642901462173|
|0.000000001     |16123          |4          |189             |0.6556488927902553|
  
The highest modularity obtained is 0.6562799279592473 at tolerance level 0.00000001 for the first run, the next smaller tolerance did not return a better modularity (diminishing returns?). The results from the first and second runs varies as expected. 
In the next run, we introduce the parameter "concurrency: 1":  
|"toleranceLevel"|"computeMillis"|"ranLevels"|"communityCount"|"modularity"      |
|:---|---:|---:|---:|---:|
|0.01            |34274          |2          |2295            |0.6524108038980835|
|0.001           |41968          |3          |238             |0.654898832773631 |
|0.0001          |41412          |3          |235             |0.6549604140236959|
|0.00001         |41472          |4          |174             |0.6550125845628862|
|0.000001        |40087          |4          |174             |0.6550125845628862|
|0.0000001       |40058          |4          |174             |0.6550125845628862|
|0.00000001      |39059          |4          |174             |0.6550125845628862|
|0.000000001     |39374          |4          |174             |0.6550125845628862|  
  
The same result was gotten from multiple runs.
With a concurrency of 1, the processing time is increased by 2-3 times, but we get stable results. Modularity is maxed out at 0.6550125845628862 with 174 communities formed and tolerance level starting at 0.00001.

Using concurreny: 1 and tolerance: 0.00001, the top communities were:
<pre>MATCH (u:User)
WHERE u.component_id = 109
RETURN u.communityId, count(*) AS num
ORDER BY num DESC limit 10</pre> 
|"u.communityId"|"num"  |
|:---|---:|
|854910         |153523 |
|886297         |128049 |
|38936          |121473 |
|404864         |116571 |
|631429         |116296 |
|695            |111812 |
|785839         |64250  |
|631747         |47073  |
|549904         |36982  |
|755579         |35362  |
  
The number of users in the top communities seems rather large, what is their relationship to each other? One thing that comes to mind could be geographical location. Using the top community_id 854910:
<pre>MATCH (n:User)-[:WROTE]->(r:Review)-[:REVIEWS]->(:Business)-[:IN_CITY]->(c:City)
WHERE n.communityId = 854910
RETURN c.city_id, count(r) AS reviews
ORDER BY reviews DESC LIMIT 10</pre>

|"c.city_id"                     |"reviews"|
|:---|---:|
|"Boston-Massachusetts-US"       |254747   |
|"Cambridge-Massachusetts-US"    |114533   |
|"Brookline-Massachusetts-US"    |105961   |
|"Somerville-Massachusetts-US"   |52942    |
|"South Boston-Massachusetts-US" |35298    |
|"Watertown-Massachusetts-US"    |28924    |
|"Quincy-Massachusetts-US"       |25933    |
|"Jamaica Plain-Massachusetts-US"|21574    |
|"Newton-Massachusetts-US"       |19656    |
|"Waltham-Massachusetts-US"      |19362    |  
  
We can see that this community is all located in the state of Massachusetts. Running the same query on community_id 886297 shows that it is all located in Texas.

### Triangle and Local Clustering Coefficient
A [Triangle](https://neo4j.com/docs/graph-data-science/current/algorithms/triangle-count/) is a set of three nodes where each node has a relationship to the other two.  
* Counts triangles in a network.
* Works on Undirected graph
* Used to compute the Local Clustering Coefficient
* Proved lower bound for triangle listing algorithm is O(n^3) or O(m^1.5), here n is the number of vertices and m is the number of edges. If you want to solve triangle counting with Matrix Multiplication method, the time complexity is same as Matrix Multiplication.
* *writeProperty*: The node property in the Neo4j database to which the triangle count is written.
* In addition to the standard execution modes there is an alpha procedure gds.alpha.triangles that can be used to list all triangles in the graph.
  
The [Local Clustering Coefficient algorithm](https://neo4j.com/docs/graph-data-science/current/algorithms/local-clustering-coefficient/) (only defined for undirected graphs) computes the local clustering coefficient for each node in the graph. The local clustering coefficient Cn of a node n describes the likelihood that the neighbours of n are also connected.

First, run the triangle algoritm for component_id 109:
<pre>CALL gds.triangleCount.write(
    'com109Graph',
    {
        writeProperty: 'triangleCount109'
    }
)
</pre>
  
This procedure completed in 2329ms, and yield 37,494,831 total triangles counted.
  
Next, calculate the local clustering coefficient:
<pre>CALL gds.localClusteringCoefficient.write(
    'com109Graph',
    {
        writeProperty: 'lcc109'
    }
)</pre>
  
The procedure completed in 4221ms.
  
Querying the triangle count and coefficient values:
<pre>MATCH (u:User) WHERE u.component_id = 109
RETURN u.name,u.triangleCount109, u.lcc109 
ORDER BY u.triangleCount109 DESC, u.lcc109 DESC LIMIT 10</pre>
|"u.name"   |"u.triangleCount109"|"u.lcc109"          |
|:---|---:|---:|
|"Steven"   |295550              |0.014583423076526433|
|"Michael"  |250168              |0.04446375837912247 |
|"Cassandra"|242354              |0.04874101041353053 |
|"Randy"    |239769              |0.02763686367489867 |
|"Dan"      |215332              |0.045442295910948866|
|"Leah"     |208813              |0.049064323129545014|
|"Phil"     |208556              |0.022058386461966218|
|"Maria"    |188912              |0.047025847637468435|
|"Guy"      |188386              |0.030417021881216255|
|"Marianne" |184200              |0.033574368867482954|
  
This tells us that the top users are friends with alot of other users, but their friends might not necessarily know each other.
Closed knitted communities can be found with high local clustering coefficient (these users are probably more likely to hang out in a group as all of them know each other):
<pre>MATCH (u:User) 
WHERE u.component_id = 109 AND u.triangleCount109 > 10
RETURN u.name,u.triangleCount109, u.lcc109 
ORDER BY  u.lcc109 DESC LIMIT 10</pre>
  
|"u.user_id"               |"u.name"  |"u.triangleCount109"|"u.lcc109"|
|:---|---:|---:|---:|
|"u-vdNG3dQk-EQMo4AoP4DSxw"|"Vinny"   |15                  |1.0       |
|"u-_-FgFe-UCtO1r4846qazDA"|"Jake"    |21                  |1.0       |
|"u-daPp3GR9Ek8YgPZDMHT81g"|"Shari"   |21                  |1.0       |
|"u-5lfHaahD4bEufAZdo84qAA"|"Samantha"|15                  |1.0       |
|"u-eQRdUkjxtSy91g8v38TokQ"|"Mike"    |15                  |1.0       |
|"u-asveGQahhSL5FWbjlE1YIg"|"Jason"   |15                  |1.0       |
|"u-rj9-ndvzOYoUIyRjTuUylA"|"Gigi"    |15                  |1.0       |
|"u-2X8Yg1sQbd_uoB_ZbI8MeA"|"Tina"    |21                  |1.0       |
|"u-NQyPXNzszWmK8WTswOx9QQ"|"Joni"    |15                  |1.0       |
|"u-BqcvOzSengNYo6XbTpwXaw"|"Tina"    |15                  |1.0       |
  
<pre>MATCH (u1:User)-[:FRIENDS]-(u2:User)
WHERE u1.component_id=109 and u2.component_id=109 and u1.user_id = "u-vdNG3dQk-EQMo4AoP4DSxw"
    RETURN u1,u2</pre>
  
![image](https://user-images.githubusercontent.com/830693/127731833-be7fcae2-c520-45b2-b3a1-6be337c4c077.png)
  
## Find important nodes (users) in a community
Of the algorithms in the Centrality category, [PageRank](https://neo4j.com/docs/graph-data-science/current/algorithms/page-rank/#algorithms-page-rank) is probably the most famous as it was used by Google Search to rank web pages in their search engine results.
* Measures importance of each node within the graph based on the number of incoming relationships and the importance of the corresponding source nodes. Underlying assumption roughly speaking is that a page is only as important as the pages that link to it.
* *dampingFactor* (defaulted 0.85): see sinks (d=1) and random restarts (d=0)
* Scales well, roughly O(log n) where n is the size of the network
* *tolerance* (defaulted 0.0000001) configuration parameter denotes the minimum change in scores between iterations
* Defined for directed (NATURAL) graphs (*orientation*: NATURAL, REVERSE, UNDIRECTED)
