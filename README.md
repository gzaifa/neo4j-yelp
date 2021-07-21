# neo4j-yelp Data Modelling and Processing
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

## Understanding yelp's domain
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
