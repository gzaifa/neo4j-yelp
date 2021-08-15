# Neo4j Yelp Data Modelling and Processing
Use Neo4J (a labelled graph database) to explore yelp data to see if any insights can be found.

Neo4j lists, as best practice, the following steps to designing the initial data model, highlighting that it is an iterative process.
<ol>
<li>Understand the domain.</li>
<li>Create high-level sample data.</li>
<li>Define specific questions for the application.</li>
<li>Identify entities.</li>
<li>Identify connections between entities.</li>
<li>Test the questions against the model.</li>
<li>Test scalability.</li>
</ol>

## Setting up files for local processing
yelp's dataset comprises of 5 json files:
<ol>
<li> yelp_academic_dataset_business.json</li>
<li> yelp_academic_dataset_checkin.json</li>
<li> yelp_academic_dataset_review.json</li>
<li> yelp_academic_dataset_tip.json</li>
<li> yelp_academic_dataset_user.json</li>
</ol>

To make processing of the files easier, I have downloaded them from [yelp](https://www.yelp.com/dataset/download) to my local file system.

In order to use the files from Neo4j desktop, they need to be placed in the imports folder. To find out where the import folder on your local drive is, go to Manage->Open Folder->Import
<br>
![image](https://user-images.githubusercontent.com/830693/126421255-a390bef3-481e-4fc8-b6c5-c22a97c3e75d.png)

This will open the import folder in a new window. To access the files from Neo4j desktop command line, you will have to enable local file import in the neo4j.conf file.  
Similarly, to access this file, go to Manage->Open Folder-> Configurations.  
Open the "neo4j.conf" file in your favorite editor and add the following configuration line:
<pre>apoc.import.file.enabled=true</pre>
<br>
After the configuration takes effect, the file can be accessed from Neo4j's Browser command line. The path is relative to the "import" folder's path, hence you would define the path as
<pre> :param yelp_business_file => '/yelp_dataset/yelp_academic_dataset_business.json'</pre>
<br> if you have unzipped the files into a "yelp_dataset" folder under "import" folder. This command sets up the path string as a param which can be access using $yelp_business_file in other commands.

## Understand the domain  
yelp is a online business directory that is used world-wide for discovering local businesses ranging from bars, restaurants, and cafes to hairdressers, spas, and gas stations. Listings are sorted by business types and results are filtered by geographical location, price, and unique features like outdoor seating, delivery service, or a the ability to accept reservations.
It publishes crowd-sourced reviews about these businesses.  
  
Some questions that came to mind:
* Do restaurants with high ratings (> 4 stars) congregate together?
* Do reviewers tend to give altogether high/low ratings? Do the majority (> 75%) of their ratings trend high/low?
* If person A gives a high rating (> 4 stars) for a restaurant, does his/her friends also gives a high rating to the same restaurant? Also low ratings.

Now that the files and Neo4j browser is set up, we can view the contents of the files on Neo4j browser.  
To view the first 10 items of the file, run the following command:
<pre>CALL apoc.load.json($yelp_business_file) YIELD value
RETURN value LIMIT 10</pre>  

For a start, I am interested in the following fields:  
1.	Business
* business_id: string, 22 character unique string business id
* name: string, the business's name
* latitude: float, latitude
* longitude: float, longitude
* stars: float, star rating, rounded to half-stars
* review_count: integer, number of reviews
* categories: an array of strings of business categories
2. User
* user_id: string, 22 character unique user id, maps to the user in user.json
* name: string, the user's first name
* review_count: integer, the number of reviews they've written
* friends: // array of strings, an array of the user's friend as user_ids
* average_stars: float, average rating of all reviews

### Importing data, creating relationships
A first pass of the fields available and using the arrows.app tool to create our first graph model.
![image](https://user-images.githubusercontent.com/830693/126449058-4722a6da-3431-4902-b483-1c157f7a0b7d.png)

Looking at this graph, I think some of the following questions can be answered (if not, we might need to refactor the graph model in a later iteration):
* How many restaurants are there?
* What is the average number of reviews that a restaurant has?
* What is the average number of reviews that each user gave?
The preceding questions looks like they would have been better answered using a traditional database, or even another NoSQL database.
The power of a labelled property graph database lies in traversing the relationships of the graph, and some of the following questions might take advantage of that:
* Find communities within the graph
* Find the important nodes (users) within the communities (Top 10)
* Find important intermediaries between communities (users that connects different communities)
	
Let's try to import "Business" and "User" datasets into our database.  
Let's see how many records there are in the datasets:
<pre>CALL apoc.load.json($yelp_business_file) YIELD value
RETURN count(value);
CALL apoc.load.json($yelp_user_file) YIELD value
RETURN count(value)</pre>
User: 2,189,457
Business: 160,585

Both apoc.load.json and "LOAD CSV" are single-threaded commands, so the loading might be able to be sped up using parallel processing with *apoc.periodis.iterate*.  
The general construct of the command is:  
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
The above command took 48709ms (around 49seconds) to run on my desktop with exactly 2,189,457 nodes created:
![image](https://user-images.githubusercontent.com/830693/126462080-da955ee7-d904-4f45-b513-44a7b1bd4ec3.png)

NOTE: Initially the command just ended without any error message but no nodes were created. To investigate further, look at the Neo4j's log files located at (you guessed it...) Manage->Open Folder->Logs.  
I found the problem under neo4j.log. There was extra curly brackets around {value} which were not required (cryptic error message was: 
<pre>The old parameter syntax `{param}` is no longer supported. Please use `$param` instead (line 1, column 59 (offset: 58))
"UNWIND $_batch AS _batch WITH _batch.value AS value  WITH {value}"</pre>

We will have to go through the user file for a second time to create the FRIEND_OF relationship between all the user nodes.
In order to make it run faster, create an index on user_id
<pre>CREATE CONSTRAINT ConstraintUniqueUserId ON (p:USER) ASSERT p.user_id IS UNIQUE</pre>
<pre>WITH 'call apoc.load.json("/yelp/yelp_dataset/yelp_academic_dataset_user.json") YIELD value RETURN value' AS load_json
CALL apoc.periodic.iterate(load_json, 'WITH value 
MATCH (u:User {user_id:value.user_id}) 
WITH u,value.friends as friends
UNWIND friends as friend
MERGE (u1:User{user_id:friend})
MERGE (u)-[:FRIEND_OF]-(u1)',
{batchSize: 1000, parallel: True, iterateList: True, retries:3}) 
YIELD batches, total
RETURN *
</pre>
NOTE: The above had trouble running to completion, after running for > 3hrs, only 18,000 relationships had been created and it was all 1-1 relationships, i.e., no single node had more than one friend as the following return 0 rows:
<pre>MATCH (p1:User)-[r:FRIEND_OF]-(p2:User)
WITH p1, count(r) as num_friends
WHERE num_friends > 1
RETURN p1,p2
</pre>

<pre>
CALL apoc.periodic.iterate("
CALL apoc.load.json('/yelp/yelp_dataset/yelp_academic_dataset_user.json')
YIELD value RETURN value
","
MERGE (u:User{id:value.user_id})
SET u += apoc.map.clean(value, ['friends','user_id'],[0]),u.name = coalesce(value.name),
	u.review_count = value.review_count,
	u.average_stars = value.average_stars
WITH u,value.friends as friends
UNWIND friends as friend
MERGE (u1:User{id:friend})
MERGE (u)-[:FRIEND_OF]-(u1)
",{batchSize: 100, iterateList: true});
</pre>

Madness!!! Am I really seeing 4,793,726,333,764 estimated rows?


Processing this line of 2 friends is successful with two relationships being created:
<pre>{"user_id":"cojecOwQJpsYDxnjtgzteQ","name":"Steven","review_count":51,"yelping_since":"2010-07-04 17:18:40","useful":136,"funny":47,"cool":44,"elite":"2010,2011","friends":"pezqbtp3BHiRUGG_Bm5_ug,wpuR1jPNjmdMEK8kXipYmQ","fans":4,"average_stars":3.79,"compliment_hot":4,"compliment_more":5,"compliment_profile":2,"compliment_cute":1,"compliment_list":0,"compliment_note":4,"compliment_plain":6,"compliment_cool":12,"compliment_funny":12,"compliment_writer":4,"compliment_photos":2}</pre>
However, increasing the number of friends in the the array results in no relationships being created:
<pre>{"user_id":"cojecOwQJpsYDxnjtgzteQ","name":"Steven","review_count":51,"yelping_since":"2010-07-04 17:18:40","useful":136,"funny":47,"cool":44,"elite":"2010,2011","friends":"_Tpd51CSlnOyvDTpOtgG5w, jVYzrVblDFSuL3GHtt8ZSA, VH18dyRNF2zrJly76eMppQ, qnYw6KaUiO4nFdfBhxqtKw, Mntyg_rQ9wgSF36wVW4asA, aUhcscNphAJQlZe1R_WRow, DVFtg_fc2FlOyOcx1Go6LQ, dboAKYo-6jWm0QkoU4Gw_A, hZMR_sCpKgrS65G3w2ufiA, 9BoPpMPWLG0xPQ1w6SDwPQ, BlYS4iE1imr8uHZUI04DPw, _rbJiH3hh_Csf_ERzSGFhA, Er1zMjQX2WxhgLWvq3LyIQ, E3dfnSs-DAQCw4Qf7J6zGg, KBZsToaFJmNMerS8gFC7Gw, pezqbtp3BHiRUGG_Bm5_ug, rdN5-4o2cqK2zC37sWSjkA, MuLxtrBiNIt0fPfQ3vM5ZQ, bn3mL61VLF_OhTYOGPb3pg, NeThj2YLGWSdlVQ-6Lk1Zw, swN__VyeFg6O_BFR0x5ZeA, GYMIDghm2k7gSTpyKNid1Q, a3H38cSs0qFsp11uRFx-ag, kJYS_6m7tYKGZLDRQv9uKg, 69GcN45awmlCq_RbLWd9uw, C2wI0o8n_CehtjafAiv7og, 6URZKBI8tRaztdo4eqMRew, s0pkzHUflmrarGSWP1G9gg, _CKhWiEVD6Xni44q4eelGw, QCNLvK6xxebYoItPU49-Og, r1nxWgnYj0WT3eY2dAsLfA, e3uXoS2ACM0ABD_uC2te9Q, v9HPVo-cqGEQ2gYSAbGnzQ, 2j-EXRubkWMhnqq6_rL-fg, jd3zDm_qigq0Ni5l_WOeLQ, aV_d1Ql5dhpTBFmiN2vkcg, UaFrZ5obw9q1XAFF6_QsBA, 17XthUbE3VBxFfKmBuV5zA, F_kM1y9821efOrLI7x0cOw, IBevAvzgKSZm-ADAkWjXFg, 0dGi59C6bWFAzrhUjm_cKQ, vV6eQMXV6jTI3b8sf1hlzA, 7luj-qkZxPaeiXUndtQSRg, RUbi_o_PihlKZaMxPwJnmQ, BmGKtjhYkFvMR13-RubrOQ, tJZmmVGVHaMqfXr-tx5w-w, aNb7imBYruQZbAqNWTFD7w, w0OabwDAGFWnWBA0w2KAFA, WvDwS_KRAQPuKNYsCdlbaw, 6dtE2xvLX32eOHEeFTBiqA, wpuR1jPNjmdMEK8kXipYmQ","fans":4,"average_stars":3.79,"compliment_hot":4,"compliment_more":5,"compliment_profile":2,"compliment_cute":1,"compliment_list":0,"compliment_note":4,"compliment_plain":6,"compliment_cool":12,"compliment_funny":12,"compliment_writer":4,"compliment_photos":2}
</pre>

