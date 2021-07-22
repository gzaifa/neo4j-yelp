#Processing user relationships
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

Tried running the above command to create FRIEND_OF relationships, however, running them in smaller batchSize, in parallel and not, all resulted in timeout after a few hours. At this point in time, I did not know what the problem was, whether it was due to wrong cypher syntax or logic or it was just too much for the system to take (I had already increased the heap size to 4GB and cache to 8GB).

#Breaking up into smaller pieces for debugging
It can be challenging to work with the big files in debugging, especially when the command runs into timeouts.  
In order to debug efficiently, I decided to split the user files into smaller ones in hopes that I can find the error.  
Using the following commands, I first parse the yelp_academic_dataset_user.json file (3.68GB) into single line json 
<pre>jq -n -c inputs yelp_academic_dataset_user_less.json > yelp_users_single_line.json</pre>
Then I use the split command to further split it into smaller files:
<pre>split -b 1000mb yelp_users_single_line.json</pre>
<pre>split -b 100mb yelp_users_single_line.json</pre>
<pre>split -b 10mb yelp_users_single_line.json</pre>
Finally I have a small enough file to be open by my editor so that I can open it and examine the contents.
