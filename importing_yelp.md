# Processing user relationships. 
![image](https://user-images.githubusercontent.com/830693/126575955-e9a6ef46-f836-4144-8621-e3a85ac4f605.png)
<pre>WITH 'call apoc.load.json("/yelp/yelp_dataset/yelp_academic_dataset_user.json") YIELD value RETURN value' AS load_json
CALL apoc.periodic.iterate(load_json, 'WITH value 
CREATE (u:User) SET
    u.user_id= value.user_id,
	u.name = coalesce(value.name),
	u.review_count = value.review_count,
	u.average_stars = value.average_stars',
{batchSize: 5000, parallel: True, iterateList: True, retries:3}) 
YIELD batches, total
RETURN *
</pre>
![image](https://user-images.githubusercontent.com/830693/126576007-fe07c3b4-3172-4ea0-97d1-02f824bd9b57.png)
Started streaming 1 records after 1 ms and completed after 41230 ms.

<pre>CALL apoc.periodic.iterate("
CALL apoc.load.json('/yelp/yelp_dataset/yelp_academic_dataset_user.json')
YIELD value RETURN value
","
MATCH (u:User{user_id:value.user_id})
WITH u,value.friends as friends
UNWIND friends as friend
MATCH (u1:User{user_id:friend})
MERGE (u)-[:FRIEND_OF]-(u1)
",{batchSize: 5000, parallel: True, iterateList: true, retries: 3})
YIELD batches, total
RETURN *
</pre>
![image](https://user-images.githubusercontent.com/830693/126576222-08082594-991a-488f-a6f6-4caca3c140bb.png)

Tried running the above command to create FRIEND_OF relationships, however, running them in smaller batchSize, in parallel and not, all resulted in timeout after a few hours. Examining the various logs in the log folder did not give me any clues... At this point in time, I did not know what the problem was, whether it was due to wrong cypher syntax or logic or it was just too much for the system to take (I had already increased the heap size to 4GB and cache to 8GB).

## Breaking up into smaller pieces for debugging
It can be challenging to work with the big files in debugging, especially when the command runs to timeouts.  
In order to debug efficiently, I decided to split the user files into smaller ones in hopes that I can find the error.  
Using the following commands, I first parse the yelp_academic_dataset_user.json file (3.68GB) into single line json 
<pre>jq -n -c inputs yelp_academic_dataset_user_less.json > yelp_users_single_line.json</pre>
Then I use the split command to further split it into smaller files:
<pre>split -b 1000mb yelp_users_single_line.json</pre>
<pre>split -b 100mb yelp_users_single_line.json</pre>
<pre>split -b 10mb yelp_users_single_line.json</pre>
Finally I have a small enough file to be open by my editor so that I can open it and examine the contents.

### Problem found
One problem found was that the UNWIND <pre>UNWIND value.friends as friend</pre> wasn't splitting the values up.
![image](https://user-images.githubusercontent.com/830693/126593085-1ac75a36-69ac-4e20-a526-25bbeb045680.png)
The solution found was to use split before unwinding.
![image](https://user-images.githubusercontent.com/830693/126593106-1250e32d-dd9c-4e19-ae3f-3c32a074c2dd.png)

## Breakthrough with Neo4j import tool
After struggling with the large amount of data and transaction requirements for a few days, I got some inspiration from [TRAN Ngoc Thach](https://thachngoctran.medium.com/exploring-yelp-dataset-with-neo4j-part-i-from-raw-data-to-nodes-and-relationships-with-python-21f52dd408ef). His strategy was to pre-process and "denormalise" the data into csv files which can be imported using Neo4J's [Import Tool](https://neo4j.com/docs/operations-manual/current/tutorial/import-tool/), which Neo4J recommends for data ingesting for more than 10 million records. (10.1 million nodes and 27million relationships)
The requirements was that the database be empty and offline while the import was running. The easiest way to do this was to specify a "new" database which the import tool will create.
The command I used:
```shell
./bin/neo4j-admin import --multiline-fields=true \
--nodes ./import/yelp_csv/area_nodes.csv \
--nodes ./import/yelp_csv/business_nodes.csv \
--nodes ./import/yelp_csv/category_nodes.csv \
--nodes ./import/yelp_csv/city_nodes.csv \
--nodes ./import/yelp_csv/country_nodes.csv \
--nodes ./import/yelp_csv/review_nodes.csv \
--nodes ./import/yelp_csv/user_nodes.csv \
--relationships ./import/yelp_csv/relationships.csv  --database=yelp
```
NOTE: Each file must be defined as its own parameter “--nodes”, or else the import would take the header from the first file and try to apply it to subsequent files.
On my machine, the import took a total of 57.623 seconds:
<pre>IMPORT DONE in 57s 623ms.
Imported:
  10987148 nodes
  27126419 relationships
  32951682 properties
Peak memory usage: 1.134GiB</pre>

The graph model which we will be referring to often:  
![Pasted Graphic](https://user-images.githubusercontent.com/830693/127666972-1cd96785-9207-4a94-adee-ff3afc8be889.png). 
