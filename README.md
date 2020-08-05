<p align="center">
    <a href="https://doi.org/10.5281/zenodo.3239509">
        <img src="https://zenodo.org/badge/DOI/10.5281/zenodo.3239509.svg" alt="DOI">
    </a>
</p>

# Wikipedia to ElasticSearch

This is a knowledge resource based on wikipedia. <br/>
But also a *multilingual* parsing mechanism that enables parsing of Wikipedia, Wikinews, Wikidata and other Wikimedia .bz2 dumps into an ElasticSearch index.

Integrated with Intel NLP Framework <a href="https://github.com/NervanaSystems/nlp-architect">NLP Architect</a>

For more information and examples check this related <a href="https://www.intel.ai/extracting-semantic-relations-using-external-knowledge-resources-with-nlp-architect/#gs.12xroe">blog post</a>.

Current supported languages: *{English, French, Spanish, German, Chinese}* <br/>

#### Extracted Relations Types and Features
*Note Relations integrity tested only for English. Other languages will probably require some tuning.<br/><br/>
3 different types of Wikipedia pages are used: {Redirect/Disambiguation/Title} in order to extract 6 different 
semantic features for tasks such as Identifying Semantic Relations, Entity Linking, Cross Document Co-Reference, Knowledge Graphs, Summarization and other.

* Redirect Links - See details at <a href="https://en.wikipedia.org/wiki/Wikipedia:Redirect">Wikipedia Redirect</a>
* Disambiguation Links - See details at <a href="https://en.wikipedia.org/wiki/Category:Disambiguation_pages">Wikipedia Disambiguation</a>
* Category Links - See details at <a href="https://en.wikipedia.org/wiki/Help:Category">Wikipedia Category</a>
* Link Title Parenthesis - See details at paper <a href="http://u.cs.biu.ac.il/~dagan/publications/ACL09%20camera%20ready.pdf">"Extracting Lexical Reference Rules from Wikipedia"</a>
* 'Is A' (extracted from page first paragraph) - See details at paper <a href="http://u.cs.biu.ac.il/~dagan/publications/ACL09%20camera%20ready.pdf">"Extracting Lexical Reference Rules from Wikipedia"</a>
* Term Frequency (TBD/WIP) - Hold a map of term frequency for computing TFIDF on Wikipadia

***

### Getting the index with Docker
This is the preferred and fastest way to get the elastic index locally.<br/>
Docker images for English contains all extracted fields. For other languages, docker images won't contain the `relations` field.

#### Docker Prerequisites
1) Before starting this one-time process, make sure docker disk image size (*in your Docker Engine Preferences*) is not limited under 150GB.
Once below one-time process done, you can decrease image size
2) If working with a Linux host, you might need to increase host vm.max_map_count,
in order to verify you have the right value (i.e 262144), run: 
        
        $>sudo sysctl vm.max_map_count
        You should see: vm.max_map_count = 262144
        If you see instead: vm.max_map_count = 65530, 
        Fix by running:
        $>sudo sysctl -w vm.max_map_count=262144
        

#### Get Image and Build Container

* Pull the latest image, use *one* of those commands (processed dump year):

    `#>docker pull aeirew/elastic-wiki` // English (August 2019) <br/>
    `#>docker pull aeirew/elastic-wiki-fr` // French (July 2020) <br/>
    `#>docker pull aeirew/elastic-wiki-es` // Spanish (July 2020) <br/>
    `#>docker pull aeirew/elastic-wiki-de` // German (July 2020) <br/>

* Once pulled, run it (all commands below use the English docker image, replace with the actual pulled image): 
   
    `#>docker run -d -p 9200:9200 -p 9300:9300 aeirew/elastic-wiki`

* The Elastic container index is still empty, all data is in a compressed file within the image,
in order to create and export the wiki data into the Elastic index (which takes a while) run the following command

    `#>docker exec <REPLACE_WITH_RUNNING_CONTAINER_ID> /tmp/build.sh`
    
* Save to new image (after done you can delete the original aeirew/elastic-wiki image, in order to save disk space)

    `#>docker commit <CONTAINER_ID> <IMAGE_NEW_NAME>`
    
* after saving stop the running container with 

    `#>docker stop <CONTAINER_ID>`

* That's it! now you can run your created image

    `#>docker run -d -p 9200:9200 -p 9300:9300 <IMAGE_NEW_NAME>`

   To test/query, you can run from terminal:
   
    `curl -XGET 'http://localhost:9200/enwiki_v3/_search?pretty=true' -H 'Content-Type: application/json' -d '{"size": 5, "query": {"match_phrase": { "title.near_match": "Alan Turing"}}}'`
    
   This should return a wikipedia page on Alan Turing.

***

### Building the index From Source

**Disclimer:** Processing English Wikipedia latest full dump (15GB .bz2 AND 66GB unpacked as .xml) including extraction of relations fields, normalization and lemmatization of text, 
will take about **5 days** (tested on MacBook pro, using stanford parser to extract relations, normalize and lemmatize the data).<br/>
In case of using this data in order to identify semantic relations between phrases at run time, It is recommended to normalize the fields for better results, thus, consider using the 'en' Docker Image to save that time. 

In case relation fields norm not needed or relations not needed all together, set normalizeFields/extractRelationFields to false in `conf.json` (as shown in "Project Configuration Files"), for a much faster data export into elastic **(< 2 hours)**.<br />

### Requisites
* Java 1.8
* ElasticSearch 6.2 (<a href=https://www.elastic.co/downloads/elasticsearch>Installation Guide</a>)
* Wikipedia xml.bz2 dump file in required language (English for example<a href=https://dumps.wikimedia.org/enwiki/latest/enwiki-latest-pages-articles.xml.bz2>download enwiki latest dump xml</a>)
* project `build.gradle` - In case of building **none** English index (and `conf.json` - `extractRelationFields=true`), remove the comment for the `stanford-model` of the required language

### Project Configuration Files
* `src/main/resources/conf.json` - basic process configuration
```
    "indexName" : "enwiki_v3" (Set your desired Elastic Search index name)  
    "docType" : "wikipage" (Set your desired Elastic Search documnent type)
    "normalizeFields" : true (Weather to run normalization of fields while processing the data)
    "extractRelationFields" : true (Weather to extract relations fields while processing the data, support only english wikipedia)
    "insertBulkSize": 100 (Number of pages to bulk insert to elastic search every iteration (found this number to give best preformence))
    "mapping" : "mapping.json" (Elastic Mapping file, should point to src/main/resources/mapping.json)
    "setting" : "en_map_settings.json" (Elastic Setting file, current support {en, fr, es, de, zh})
    "host" : "localhost" (Elastic host, were Elastic instance is installed and running)
    "port" : 9200 (Elastic port, host port were Elastic is installed and running, elastic defualt is set to 9200)
    "wikiDump" : "dumps/enwiki-latest-pages-articles.xml.bz2" (WikiMedia .bz2 downloaded dump file location)
    "scheme" : "http" (Elastic host schema, should probebly stay unchanged)
    "shards" : 1 (Number of Elastic shards to use)
    "replicas" : 0 (Number of Elastic replicas to use)
    "lang": "en" (current support {en, fr, es, de, zh})
```
* `src/main/resources/mapping.json` - Elastic wiki index mapping (Should probably stay unchanged)
* `src/main/resources/{en,es,fr,de,zh}_map_settings.json` - Elastic index settings (Should probably stay unchanged)
* `src/main/resources/lang/{en,es,fr,de,zh}.json` - language specific configuration files
* `src/main/resources/stop_words/{en,es,fr,de,zh}.txt` - language specific stop-words list

* Make sure Elastic process is running and active on your host (if running Elastic locally your IP is <a href="http://localhost:9200/">http://localhost:9200/</a>)
* Checkout/Clone the repository
* Put wiki xml.bz2 dump file (no need to extract the bz2 file!) in: `dumps` folder under root checkout repository.<br/> 
<i><b>Recommendation:</b> Start with a small wiki dump, make sure you like what you get (or modify configurations to meet your needs) before moving to a full blown 15GB dump export..</i>
* Make sure `conf.json` configuration for Elastic are set as expected (default localhost:9200)
* From command line navigate to project root directory and run:<br/>
`./gradlew clean build -x test` <br/>
*Should get a message saying: `BUILD SUCCESSFUL in 7s`*
* Extract the build zip file created at this location `build/distributions/WikipediaToElastic-1.0.zip`
* Run the process from command line:<br/>
`java -Xmx6000m -DentityExpansionLimit=2147480000 -DtotalEntitySizeLimit=2147480000 -Djdk.xml.totalEntitySizeLimit=2147480000 -jar build/distributions/WikipediaToElastic-1.0/WikipediaToElastic-1.0.jar`

* To test/query, you can run from terminal:<br/>
`curl -XGET 'http://localhost:9200/enwiki_v3/_search?pretty=true' -H 'Content-Type: application/json' -d '{"size": 5, "query": {"match_phrase": { "title.near_match": "Alan Turing"}}}'`

This should return a wikipedia page on Alan Turing.

***

### Elastic Page Query

Once process is complete, two main query options are available (for more details and title query options, see `mapping.json`):<br/>
* title.plain - fuzzy search (sorted)
* title.keyword - exact match


***

### Generated Elastic Page Example

Pages that have been created with the following structures (also see "Created Fields Attributes" for more details):   

**Page Example (Extracted from Wikipedia disambiguation page):**
```json
{
  "_index": "enwiki_v3",
  "_type": "wikipage",
  "_id": "40573",
  "_version": 1,
  "_score": 20.925367,
  "_source": {
    "title": "NLP",
    "text": "{{wiktionary|NLP}}\n\n'''NLP''' may refer to:\n\n; .....",
    "relations": {
      "isPartName": false,
      "isDisambiguation": true,
      "disambiguationLinks": [
        "Natural language programming",
        "New Labour",
        "National Library of the Philippines",
        "Neuro linguistic programming",
        "Natural language processing",
        "National Liberal Party",
        "Natural Law Party",
        "National Labour Party",
        "Normal link pulses",
        "New Labour Party"
      ],
      "categories": [
        "disambiguation"
      ],
      "titleParenthesis": [],
      "disambiguationLinksNorm": [
        "natural language processing",
        "natural law party",
        "national labour party",
        "natural language programming",
        "national library philippine",
        "neuro linguistic programming",
        "new labour party",
        "national liberal party",
        "new labour",
        "normal link pulse"
      ],
      "categoriesNorm": [
        "disambiguation"
      ],
      "titleParenthesisNorm": []
    }
  }
}
 ```
**Page Example (Extracted from Wikipedia redirect page):**
```json
{
  "_index": "enwiki_v3",
  "_type": "wikipage",
  "_id": "2577248",
  "_version": 1,
  "_score": 20.925367,
  "_source": {
    "title": "Nlp",
    "text": "#REDIRECT",
    "redirectTitle": "NLP",
    "relations": {
      "isPartName": false,
      "isDisambiguation": false
    }
  }
}
 ```

### Created Fields Attributes 

| json field  | Value | comment |
| ------------- | ------------- | ------------- |
| _id | Text | Wikipedia page id |
| _source.title | Text | Wikipedia page title |
| _source.text | Text | Wikipedia page text |
| _source.redirectTitle | Text (optional) | Wikipedia page redirect title |
| _source.relations.beCompRelations | List (optional) | Be-Comp (or 'Is A') relation list |
| _source.relations.beCompRelationsNorm | List (optional) | Be-Comp (or 'Is A') relation list norm |
| _source.relations.categories | List (optional) | Categories relation list |
| _source.relations.categoriesNorm | List (optional) | Categories relation list norm |
| _source.relations.isDisambiguation | Bool (optional) | is Wikipedia disambiguation page |
| _source.relations.isPartName | List (optional) | is Wikipedia page name description |
| _source.relations.titleParenthesis | List (optional) | List of disambiguation secondary links  |
| _source.relations.titleParenthesisNorm | List (optional) | List of disambiguation secondary links norm |

### Citing
If you use this project or the created data in your research, please use the following citation https://doi.org/10.5281/zenodo.3239509

