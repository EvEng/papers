Graph databases
===============

Certain tasks may require queries that travers a network of
links. Graph databases are supposed to help.

linux.md has the info on extracting data from linux kernel git
repository. The extracted files are on da2 container /data/linux .


The following is the command to run docker container on any host
that has docker engine

```
docker run -d \
    -p 7474:7474 -p 7687:7687 \
    -v $(pwd)/neo4j/data:/data \
    -v $(pwd)/neo4j/logs:/logs \
    -v $(pwd):/import \
    --name yourpreferedname \
    neo4j:3.0
```
The data will be residing on $(pwd) which will appear as /import
within the container. If it complains, you might need to remove existing container via
```
docker rm -f yourpreferedname
```

To connect to a container:
```
docker exec -it yourpreferedname bash
```

Once connected we can preapare linux commit/maintainer data for
import
```
cd /import
(echo "Author:ID(A),n:string";gunzip -c linux.delta.gz | cut -d\; -f5 |  sed 's/,/ /g;s/;/,/g' | iconv -ct ASCII//TRANSLIT|  sed 's/,/ /g;s/;/,/g'  | sort -u| grep -v '^\s*$') > author.csv
(echo "Email:ID(E)";gunzip -c linux.delta.gz | cut -d\; -f7 |  sed 's/,/ /g;s/;/,/g' | sort -u) > email.csv
(echo "Commit:ID(C)"; gunzip -c linux.delta.gz | \
   cut -d\; -f2,4 |perl -ane 's/;/\n/g;print' | sort -u| grep -v '^$' | awk '{print $0}') > commit.csv
(echo ":START_ID(A),:END_ID(E)";gunzip -c linux.delta.gz | cut -d\; -f5,7  |iconv -ct ASCII//TRANSLIT |  sed 's/,/ /g;s/;/,/g'  | sort -u| grep -v '^\s*,'| grep -v ',$') > a2e.csv
(echo ":START_ID(C),:END_ID(A)";gunzip -c linux.delta.gz | cut -d\; -f2,5 | iconv -ct ASCII//TRANSLIT |  sed 's/,/ /g;s/;/,/g' | sort -u| grep -v ',\s*$') > c2a.csv
(echo ":START_ID(C),:END_ID(C)"; gunzip -c linux.delta.gz | cut -d\; -f2,4 | \
  sed 's/,/ /g;s/;/,/g' | sort -u | grep --binary-files=text  -v ',$') > parent.csv
```
  
Once the data is prepared: import and replace the database:
  
```
/var/lib/neo4j/bin/neo4j-import --into /var/lib/neo4j/data/databases/linux.db --id-type string \
  --nodes:C commit.csv --nodes:A author.csv  --nodes:E email.csv \
  --relationships:PARENT parent.csv --relationships:AUTHOR c2a.csv \
  --relationships:EMAIL a2e.csv

rm -rf /var/lib/neo4j/data/databases/graph.db
mv /var/lib/neo4j/data/databases/linux.db /var/lib/neo4j/data/databases/graph.db
```
After that you need to remove the neo4j container and restart exactly as described above.

Probably you need to remove it before you call the neo4j-import command. If it still doesn't work, you have 2 options: (1) just rename it:
(2)if you really want to remove that graph.db,do this:
```
docker run -d \
    -p 7474:7474 -p 7687:7687 \
    -v $(pwd)/neo4j/data:/data \
    -v $(pwd)/neo4j/logs:/logs \
    -v $(pwd):/import \
    --name yourpreferedname \
    neo4j:3.0 -it /bin/bash
rm -rf /var/lib/neo4j/data/databases/graph.db
```
instead of calling the first command.

```
mv /var/lib/neo4j/data/databases/linux.db /var/lib/neo4j/data/databases/graph.db
```

Now container needs to be restarted for the changes to take effect.


# CYPHER examples

A fairly good intro to Cypher is [the official one](https://neo4j.com/developer/cypher-query-language/)


Now lets run some queries, first a minor contributor:
```
MATCH (c:C)-[r:AUTHOR]->(n:A) where (n.Author='A Raghavendra Rao') RETURN c,r,n;
```


Lets do something more sophisticated, for example developer
succession measure based on the authors in parent commits:
```
MATCH (mentors:A)<-[r1:AUTHOR]-(p:C)<-[pr:PARENT]-(c:C)-[r:AUTHOR]->(n:A) where (n.Author='AKASHI Takahiro' and not n = mentors)
return mentors, count(c.Commit) AS ncommits
ORDER BY ncommits DESC
LIMIT 25;
```

The second most likely mentor for AKASHI is Linus:


Mentors        | NCommits
-------------- | --------
James Morse    | 3
Linus Torvalds | 2
Li Bin         | 1


We can consider as mentors not just authors of an immediate parent but up to 30
parent commits away (*1..30) or we can simply add * for an arbitrary number of parents away. 
~~~~
MATCH (mentors:A)<-[r1:AUTHOR]-(p:C)<-[:PARENT*1..30]-(c:C)-[r:AUTHOR]->(n:A) 
 where (n.Author='AKASHI Takahiro' and not n = mentors)
 return mentors, count(c.Commit) AS ncommits
 ORDER BY ncommits DESC
LIMIT 25;
~~~~
The result is quite different:

Mentors        | NCommits
-------------- | --------
Will Deacon    | 67
Laura Abbott   | 44
Andre Przywara | 36


Multiple relationships can also be cobined (example is from the tutorial) with variables added
for output or where clause, as well as conditions imposed, for example recency:
```
    relationship-types like -[:KNOWS|:LIKE]->
    a variable name -[rel:KNOWS]-> before the colon
    additional properties -[{since:2010}]->
    structural information for paths of variable length -[:KNOWS*..4]->
```
