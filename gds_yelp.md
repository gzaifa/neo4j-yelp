# Neo4j's Graph Data Science (gds) library  
Neo4j's gds library contains graph algorithms which are divided into categories which represent different problem classes.  
| Category      | Description | Algorithms    |
| :---        |    :----   |          --- |
| Centrality      | Are used to determine the importance of distinct nodes in a network.       | Page Rank, Article Rank, Eigenvector Centrality, Betweeness Centrality, Degree Centrality   |
| Community Detection   | Are used to evaluate how groups of nodes are clustered or partitioned, as well as their tendency to strengthen or break apart.         | Louvain, Label Propagation, Weakly Connected Components, Triangle Count, Local Clustering Coefficient      |
| Path finding       | Are used to find the shortest path between two or more nodes or evaluate the availability and quality of paths. | Dijkstra Source-Target, Dijkstra Single-Source, A*, Yen’s algorithm |
| Similarity | Are used to compute the similarity of pairs of different vector-based metrics.  | Node similarity,K-Nearest Neighbors  |

Ensure that the Graph Data Science Library is installed. It can be found under *Plugins*
![image](https://user-images.githubusercontent.com/830693/127670097-0c59aca4-7d29-493e-b198-daa52caa0d68.png)  

Some scenarios which I want to run with gds:
<ol>
<li>Find communities within the graph</li>
<li>For these communities (or one of the communities), find the top 10 important nodes</li>
<li>Find in-between nodes which acts as the "bridge" between different communities</li>
</ol>

## Find communities within the graph  
What are the properties that we want in a community:  
* Many edges connecting vertices within a community  
* More connections internally then externally
 
