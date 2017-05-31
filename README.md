GraphAware NLP Using OpenNLP
==========================================

Getting the Software
---------------------

### Server Mode
When using Neo4j in the <a href="http://docs.neo4j.org/chunked/stable/server-installation.html" target="_blank">standalone server</a> mode, you will need the <a href="https://github.com/graphaware/neo4j-framework" target="_blank">GraphAware Neo4j Framework</a> and <a href="https://github.com/graphaware/neo4j-nlp" target="_blank">GraphAware NLP</a> .jar files (both of which you can download here) dropped into the plugins directory of your Neo4j installation. Finally, the following needs to be appended to the `neo4j.conf` file in the `config/` directory:

```
  dbms.unmanaged_extension_classes=com.graphaware.server=/graphaware
  com.graphaware.runtime.enabled=true

  com.graphaware.module.NLP.2=com.graphaware.nlp.module.NLPBootstrapper
```

### For Developers
This package is an extention of the <a href="https://github.com/graphaware/neo4j-nlp" target="_blank">GraphAware NLP</a>, which therefore needs to be packaged and installed beforehand. No other dependencies required.

```
  cd neo4j-nlp
  mvn clean install

  cd ../neo4j-nlp-opennlp
  mvn clean package
```


Introduction and How-To
-------------------------

The Apache OpenNLP library provides basic features for processing natural language text: sentence segmentation, tokenization, lemmatization, part-of-speach tagging, named entities identification, chunking, parsing and sentiment analysis. OpenNLP is implemented by extending the general <a href="https://github.com/graphaware/neo4j-nlp" target="_blank">GraphAware NLP</a> package with extra parameters:

### Tag Extraction / Annotations
```
#Annotate the news
MATCH (n:News)
CALL ga.nlp.annotate({text:n.text, id: n.uuid, textProcessor: "com.graphaware.nlp.processor.opennlp.OpenNLPTextProcessor", pipeline: "tokenizer"}) YIELD result
MERGE (n)-[:HAS_ANNOTATED_TEXT]->(result)
RETURN n, result
```

Available parameters are:
  * the same ones as described in <a href="https://github.com/graphaware/neo4j-nlp" target="_blank">parent class GraphAware NLP</a>
  * `sentimentProbabilityThr` (optional, default *0.7*): if assigned sentiment label has confidence smaller thank this threshold, set sentiment to *Neutral*
  * `customProject` (optional): add user trained/provided models associated with specified project, see paragraph *Customizing pipeline models*

Available pipelines:
  * `tokenizer` - tokenization, lemmatization, stop-words removal, part-of-speach tagging (POS), named entity recognition (NER)
  * `sentiment` - tokenization, sentiment analysis
  * `tokenizerAndSentiment` - tokenization, lemmatization, stop-words removal, POS tagging, NER, sentiment analysis
  * `phrase` (not supported yet) - tokenization, stop-words removal, relations, sentiment analysis

### Sentiment Analysis
The current implementation of a sentiment analysis is just a toy - it relies on a file with 100 labeled twitter samples which are used to build a model when Neo4j starts (general recommendation for number of training samples is 10k and more). The current model supports only three options - Positive, Neutral, Negative - which are chosen based on the highest probability (the algorithm returns an array of probabilities for each category). If the highest probability is less then 70% (default value which can be customized by using parameter *sentimentProbabilityThr*), the category is not regarded trustworthy and is set to Neutral instead.

The sentiment analysis can be run either as part of the annotation (see paragraph above) or as an independent procedure (see command below) which takes in AnnotatedText nodes, analyzes all attached sentences and adds to them a label corresponding to its sentiment.

```
MATCH (a:AnnotatedText {id: {id}})
CALL ga.nlp.sentiment({node:a, textProcessor: "com.graphaware.nlp.processor.opennlp.OpenNLPTextProcessor"}) YIELD result
MATCH (result)-[:CONTAINS_SENTENCE]->(s:Sentence) 
RETURN labels(s) as labels
```

### Customizing pipeline models
To add new customized model (currenlty NER and Sentiment), one can do it via Cypher:
```
CALL ga.nlp.train({[project: "myXYProject",] alg: "NER", model: "component", file: "<path_to_your_training_file>" [, lang: "en", ...<training-parameters-of-choice>...]})
```
  * `project` (case insensitive) is an arbitrary string that allows to specify in the `annotate()` procedure that we want, apart from the default models, to use also the custom model(s) associated with `project`
  * `alg` (case insensitive) specifies which algorithm is about to be trained; currently available algs: `NER`, `sentiment`
  * `model` arbitrary string that provides, in combination with `alg` (and with `project` if it's specified), a unique identifier of the model that you want to train (will be used for e.g. saving it into .bin file)
  * `file` is path to the training data file
  * `lang` (default is "en") specifies the language
  * **training parameters** are optional and defined in `com.graphaware.nlp.util.GenericModelParameters` and are not universal (some might be specific to only certain Text Processor):
    * *iter* - number of iterations
    * *cutoff* - useful for reducing the size of n-gram models, it's a threashold for n-gram occurrences/frequences in the training dataset
    * *threads* - provides support for multi-threading
    * *entityType* - name type to use for NER training, by default all entities (classes such as "Person", "Date", ...) present in provided training file are used
    * *nFolds* - parameter for cross-validation procedure (default is 10), see paragraph *Validation*
    * *trainerAlg* - specific for OpenNLP
    * *trainerType* - specific for OpenNLP

The trained model is saved to a binary file in Neo4j's `import/` directory with name format: `<lang>-<alg>-<model>-<project>.bin`. Each time Neo4j starts, this directory is scanned for files (models) in this format, i.e. you don't need to train the same model again when you restart Neo4j.
  * `NER` - default models (Person, Location, Organization, Date, Time, Money, Percentage) plus all registered customized models (assigned to specified `project`) are used when invoking `ga.nlp.annotate()` (see example below)
  * `Sentiment` - sentiment analysis is run only once (user-trained one has a priority over the default one); if more custom models are provided, the latest one (or the last one in the list in `import/` directory) is used

**Usage of new models:**
```
# Example of a text to analyze
CREATE (l:Lesson {lesson: "Power system distribution at Kennedy Space Center (KSC) consists primarily of high-voltage, underground cables. These cables include approximately 5000 splices.ľ Splice failures result in arc flash events that are extremely hazardous to personnel in the vicinity of the arc flash. Some construction and maintenance tasks cannot be performed effectively in the required personal protective equipment (PPE), and de-energizing the cables is not feasible due to cost, lost productivity, and safety risk to others implementing the required outages. To verify alternate and effective mitigations, arc flash testing was conducted in a controlled environment. The arc flash effects were greater than expected. Testing also demonstrated the addition of neutral grounding resistors (NGRs) would result in substantial reductions to arc flash effects. As a result, NGRs are being installed on KSC primary substation transformers. The presence of the NGRs, enable usage of less cumbersome PPE.  Laboratory testing revealed higher than anticipated safety risks from a potential arc-flash event in a manhole environment when conducted at KSCęs unreduced fault current levels.ľ The safety risks included bright flash, excessive sound, and smoke.ľľ Due to these findings and absence of other mitigations installed at the time, manhole entries require full arc-flash PPE.ľ Furthermore, manhole entries were temporarily restricted to short duration inspections until further mitigations could be implemented.ľ With installation of neutral grounding resistors (NGRs) on substation transformers, the flash, sound and flame energy was reduced.ľ The hazard reduction was so substantial that the required PPE would be less cumbersome and enable effective performance of maintenance tasks in the energized configuration."})

WITH l

# Annotate it and use newly trained NER model(s)
CALL ga.nlp.annotate({text:l.lesson, id: l.uuid, customProject: "myXYProject"}) YIELD result
MERGE (l)-[:HAS_ANNOTATED_TEXT]->(result)
RETURN l, result;
```

**Format of training datasets:**
  * `NER` - one sentence per line, one empty line between two different texts (paragraphs); there must be a space before and after each `<START:my_category>` and `<END>` statement; example for two new categories ("component" and "tool"):
    ```
    Implementing  <START:tool> remote-controlled pressure regulators <END>  and  <START:tool> digital pressure gauges <END>  in the designs of new systems would improve launch processing in future programs because they would prevent launch delays and provide for safer operations.
    With the capability to adjust pressures remotely, crews would not have to be sent into the field during hazardous operations.
    With  <START:tool> digital pressure gauges <END> , differences between local and remote pressure readings would be eliminated, and there would be no need to reconcile the differences.

    During the Space Shuttle STS-80 mission, the two planned Extravehicular Activities (EVA) were scrubbed when the astronauts discovered that the outer <START:component> airlock hatch <END> would not open.
    This impacted the plan to evaluate the EVA tools that would be used for the construction and maintenance of the International Space Station.
    Upon return from the mission, troubleshooting revealed that the  <START:component> airlock actuator <END>  that is used for latching and unlatching the hatch was defective.
    This  <START:component> actuator <END>  was removed and replaced.
    ```
  * `sentiment` - two columns separated by a white space (tab): the first column is a category as integer (0=VeryNegative, 1=Negative, 2=Neutral, 3=Positive, 4=VeryPositive), the second column is a sentence; example:
    ```
    3   Watching a nice movie
    1   The painting is ugly, will return it tomorrow...
    3   One of the best soccer games, worth seeing it
    3   Very tasty, not only for vegetarians
    1   Damn..the train is late again...
    ```

**Validation:**
Model validations are performed using OpenNLP cross-validation method. The validation runs *n*-fold times on the same training file, but each time selecting different set of trainig and testing data with the ratio of *train:test = (n-1):1*. Validation measures (Precision, Recall, F-Measure) are pooled together and returned to the user as a result.