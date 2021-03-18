## Build a RDF Knowledge Graph from CSV files

This short tutorial guides you through building a RDF Knowledge Graph about restaurants and cuisines from [2 CSV sample files generated from a dataset found on Kaggle](https://data.world/mgarfield/restaurants/), using YARRRML mappings. 

> This tutorial is accessible to anyone without the need to understand anything else than reading English, just follow the instructions and copy/paste the text at the right place in a website.

## Choose a data model

**Choosing the concepts and properties** for your Knowledge Graph from existing and recognized ontologies is the most important part of the process. Using existing ontologies connects your graph to existing data published using the same ontologies. Using your own URIs for properties and entities types makes your RDF Knowledge Graph isolated, and completely missed the point of build a knowledge graph.

In this case I followed those steps to define my Knowledge Graph concepts:

* [Schema.org](https://schema.org/) is a good vocabulary built by Google which cover a large amount of concepts and properties. It is the standard vocabulary used by Search Engines, such as Google search, to understand metadata. It covers most of the entities and properties we want to create for the columns of our CSV files. 
* But the "Cuisine" concept does not exist in Schema.org: the `schema:cuisine` points to a text, but we want to create a new entity for cuisines, to better connect them). So we will [search "food ontology cuisine" in Google](https://www.google.com/search?q=food+ontology+cuisine), to find a more specific ontology about food. We find the `fo:Cuisine` concept from the [Food Ontology published by the BBC](https://www.bbc.co.uk/ontologies/fo#terms_cuisine). 
* Another interesting ontology would have been [FoodON](https://foodon.org/), which is really detailed and well connected to other ontologies, but in our simple use-case the Cuisine concept from the BBC Food Ontology is good enough.

## Convert the data to a RDF knowledge graph

1. **Go to [https://rml.io/yarrrml/matey 🦜](https://rml.io/yarrrml/matey)**

2. **Create 2 new datasources** for the [`dataworld-restaurants.csv`](/dataworld-restaurants.csv) and [`dataworld-cuisines.csv`](/dataworld-cuisines.csv) files. Click on the **Input:** tab on the left of the Matey web UI to create new datasources.
* [`dataworld-restaurants.csv`](/dataworld-restaurants.csv):

```
Restaurant ID,Restaurant Name,City,country name,Address,Locality,Locality Verbose,Longitude,Latitude,Cuisines,Average Cost for two,Currency,Has Table booking,Has Online delivery,Is delivering now,Switch to order menu,Price range,Aggregate rating,Rating color,Rating text,Votes
53,Amber,New Delhi,India,"N-19, Connaught Place, New Delhi",Connaught Place,"Connaught Place, New Delhi",77.220891,28.630197,indian,1800,Indian Rupees(Rs.),Yes,Yes,No,No,3,2.6,Orange,Average,152
55,Berco's,New Delhi,India,"G-2/43, Middle Circle, Connaught Place, New Delhi",Connaught Place,"Connaught Place, New Delhi",77.217298,28.632452,Chinese,1100,Indian Rupees(Rs.),Yes,Yes,No,No,3,3.9,Yellow,Good,2639
60,Colonel's Kababz,New Delhi,India,"29, Defence Colony Market, Defence Colony, New Delhi",Defence Colony,"Defence Colony, New Delhi",77.230591,28.574036,indian,900,Indian Rupees(Rs.),Yes,No,No,No,2,3.2,Orange,Average,600
```

* [`dataworld-cuisines.csv`](/dataworld-cuisines.csv):

```
cuisines,diets
Chinese,vegetarian|low_lactose_diet
indian,halal|low_lactose_diet|vegetarian
```

3. Copy/paste those [YARRRML mappings](/mappings.yarrr.yml) in the **input: YARRRML** box in the middle of the Matey web UI

```yaml
prefixes:
  grel: "http://users.ugent.be/~bjdmeest/function/grel.ttl#"
  rdfs: "http://www.w3.org/2000/01/rdf-schema#"
  schema: "https://schema.org/"
  fo: "http://purl.org/ontology/fo/"
  mykg: "https://w3id.org/mykg/"
  # We use w3id.org to simulate persistent ID for our entities
mappings:
  restaurants:
    sources:
      - ['dataworld-restaurants.csv~csv']
    # We define the subject and graph of this mapping
    s: mykg:restaurant/$(Restaurant ID)
    g: mykg:graph/restaurants
    # Then we define the predicate/objects for this subject
    # aka. the properties/values for this entity
    po:
      - [a, schema:Restaurant]
      - [rdfs:label, $(Restaurant Name)]
      # We link to cuisine here by creating the same URI (identifier):
      - [schema:servesCuisine, mykg:cuisine/$(Cuisines)~iri]
      - [schema:location, $(country name)]
      - [schema:address, $(Address)]
      - [schema:priceRange, $(Price range)]
      - [schema:currenciesAccepted, $(Currency)]
      - [schema:acceptsReservations, $(Has Table booking)]
      - [schema:aggregateRating, $(Aggregate rating)]
  cuisines:
    sources:
      - ['dataworld-cuisines.csv~csv']
    s: mykg:cuisine/$(cuisines)
    g: mykg:graph/cuisines
    po:
      - [a, fo:Cuisine]
      - [rdfs:label, $(cuisines)]
      # We use a function to split the "diets" cells using |
      - p: schema:suitableForDiet
        o:
          function: grel:string_split
          parameters:
          - [grel:p_string_sep, "\|"]
          - [grel:valueParameter, $(diets)]
```

Note that we use our own `mykg:` namespace for the Knowledge Graph entities URIs (the restaurants and cuisines entities generated). But we use already existing ontology terms for the properties and entity classes.

We used the free [w3id.org](https://w3id.org/) persistent ID system to define our `mykg:` URI. We did not created a new entry in w3id.org since this is an example, but this can be easily done with a simple [pull request to their GitHub repository](https://github.com/perma-id/w3id.org).

4. Click on **Generate RML**, then **Generate LD** buttons. The conversion should take around 10s.

Here is a screenshot of what you should see after converting the data, the RDF Knowledge Graph has been generated in the **Output: Turtle/TriG** panel on the right.

![Screenshot of Matey Web UI](screenshot-matey-web-ui.png)

## Publish the RDF Knowledge Graph

To be able to query your RDF Knowledge Graph, you will need to host it a triplestore (a database for RDF data). There are a lot of different triplestores out there, most of them allow to expose a SPARQL endpoint which allow anyone to query your RDF Knowledge Graph using the SPARQL query language. 

* Host your own triplestore: if you have access to a server, and are savvy enough, you can easily deploy a triplestore for free such as [OpenLink Virtuoso](http://vos.openlinksw.com/owiki/wiki/VOS), [Blazegraph](https://github.com/blazegraph/database), or [Ontotext GraphDB](https://graphdb.ontotext.com/), and load your RDF Knowledge graph in it.

* Another solution, for small pieces of RDF data, would be to publish your RDF as [Nanopublications](http://nanopub.org/wordpress/) which is a decentralized network of triplestores using encrypted keys and [ORCID accounts](https://orcid.org/) to authenticate publishers. This can be done easily with the [nanopub-java](https://github.com/Nanopublication/nanopub-java) library, or the [nanopub Python](https://pypi.org/project/nanopub/) package.
* You can also use cloud service provider to publish and expose RDF Knowledge Graphs, such as [Amazon Neptune](https://aws.amazon.com/neptune/) or [Dydra](https://dydra.com/login)

