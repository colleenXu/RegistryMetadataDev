# Advantages:


## reduces total number of queries and redundant queries

Based on this definition: an operation/metaKG edge is a unique combo of input (Biolink semantic) type, output type, and association data (so different knowledge sources can distinguish two operations)

### Example: 

Monarch (Biolink API) has a lot of useful info. However, BTE currently uses a very narrow set of x-bte operations to get data from this resource. 

One problem is that the current x-bte has to write multiple operations for a query (like PhenotypicFeature -> Disease) to handle each ID namespace Monarch gives as input and output. 

A related problem is that Monarch will resolve IDs internally and return the same data for different IDs (of the same entity) - so BTE would only need to query once. But with multiple operations written to handle the ID namespaces, BTE will query multiple times and won't know that the API is returning the same info each time. This redundancy could cause issues with scoring based on number of associations/edges.
  
**How new x-bte solves this:**

- new x-bte can have 1 operation/metaKG edge for 1 Monarch query between semantic types like PhenotypicFeature -> Disease
- it can handle querying Monarch's PhenotypicFeature -> Disease query with an HP **or** EFO **or** UMLS **or** MP ID. 
  - inside one operation, the input ID namespaces (and how to query) is a list. When there's > 1 element in this list, BTE should iterate over it. When BTE encounters the element with an ID namespace it has an ID for, BTE should use its data to do 1 query for this operation.
- it can handle HGNC, MGI, FlyBase, and other IDs in the output.
  - inside one operation, the output ID namespaces (and their corresponding response field) is a list. It still expects 1 ID namespace per response field (this may involve custom prior processing by BTE). When there's > 1 element in this list, BTE should iterate over it for each result, looking for the response field. When BTE finds a response field/ID namespace that is in the result, it can proceed with that ID to the ID resolution step.


## structure adds useful metadata to response field mapping (context, interpreting variable values)

### Example 1 

"Context" is something that could be useful, for querying, interpreting results, and scoring results. It also comes up in querying and interpreting clinical data. 

However, there's no way in the current x-bte to annotate a response field as context or give info on how to interpret it (is it a species-specific context, an experiment-specific context like the cell line used). 

### Example 2

[DISEASES](https://diseases.jensenlab.org/Search)'s [gene-disease associations] in our [pending BioThings API](http://pending.biothings.io/DISEASES) include numeric (like z-score) and categorical variables (like evidence type and confidence ranking) that could be useful to users for interpreting results and scoring. 

However, these variables need additional info to be understood: range, possible values (categorical variables), what a direction means, and where to learn more about this variable and its calculation. 

*Note: I found info that suggests that this resource updates weekly, which means we could update our API...Another resource from this lab that we may want to bring into Translator is [TISSUES](https://tissues.jensenlab.org/About) (gene-tissue relationships for humans and other animals).*


**How new x-bte solves these:**

- new x-bte groups info (static or mapped response fields) to main categories: inputsAndQueryInfo, outputs, predicateInfo, references, provenances, **contexts, categoricalVariables, numericVariables**, and otherProperties. 
- contexts maps each response field to a type (key) like taxonSpecific, diseaseSpecific, cohortSpecific, experimentalSpecific. Additional information could be added like expected values, ID namespace, etc (although that's not fleshed out in the schema yet)
- numericVariables and categoricalVariables allow additional info to be added along with the mapped response field
- **while the TRAPI standard is evolving, we have a flexible but useful structure for our metadata to "write once" into.** Then as the TRAPI standard changes, we only need to change the code for transforming into the TRAPI standard. 


**Additional perk:**
the structure also makes it easier for the x-bte writer to see what kind of defined info/categories to map the response fields to, rather than using arbitrary names for the mapping. It should also make the process of annotating response fields easier.


## Built-in automated testing

Automated testing with a known input ID and output ID could help us see if the endpoint is working, if there is an issue with processing and transforming the API output to TRAPI, or an update had a major data change and/or parser issue. 

**How new x-bte solves this:**
- The testExamples section for each operation! This is an array (since some operations may have multiple input / output ID types to test). The basic test is a known input CURIE and one output CURIE that is supposed to be in the response (using the prefixes Translator uses). 
- In some cases, there is no testExamples section due to issues finding an example (happens with APIs we don't own) or the operation retrieving no associations (happens with auto-generated x-bte annotation). We may consider removing those operations...
- The test examples are static and hand-curated by CX. If a test fails because the data changed drastically, the failure would alert the developer to review what changed with the data. If everything is fine, they could update the test. 


## website template for link-outs

Users and NCATS members have expressed interest in having link-outs from the associations/edges returned to the original knowledge sources. However, the current x-bte doesn't support this, and these website URLs are almost always NOT in the raw API response.

**How new x-bte solves this:**
- under the references section for each operation, there is a websiteTemplate set of metadata. This can include a template string for generating the website url from information like the input ID, output ID, or the value of other response fields...which would be specific for that particular association/edge/result. 
- Kevin and I discussed having a "dictionary" to map the fields of the template to the input ID / output ID / other response fields. Another solution may be to try Jinja for templating.



### work involved regardless of which x-bte is there

getting current edge attributes / node attributes compliant with trapi

some more curation to annotate more info inside the raw api response
- would involve mapping more relation/source/publications etc. to the api response

making sure BTE can correctly transform those new info in response to trapi