# drugindication_ml

Sources
-------
1- Drugbank 
RDF Dataset Obtained from: http://download.bio2rdf.org/release/4/drugbank/drugbank.nq.gz
Original complete Dataset Obtained from : https://www.drugbank.ca/releases/latest

2-Kegg
RDF Dataset Obtained from: http://download.bio2rdf.org/release/4/kegg/kegg-drug.nq.gz
		       http://download.bio2rdf.org/release/4/kegg/kegg-genes.nq.gz 
 
Original complete Dataset Obtained from : ftp://ftp.genome.jp/pub/kegg/

3- SIDER 
RDF Dataset Obtained from: http://download.bio2rdf.org/release/4/sider/sider-se.nq.gz
Original complete Dataset Obtained from:  http://sideeffects.embl.de/media/download/

4- NDF-RT 

Original complete Dataset Obtained from: http://evs.nci.nih.gov/ftp1/NDF-RT/NDF-RT_XML.zip
 
5- MEDDRA

RDF Dataset Obtained from: http://purl.bioontology.org/ontology/MEDDRA

6- Drug Indication from NDR-FT and DrugCentral

RDF Data generated: data/input/unified-gold-standard-umls.nq.gz

------------------------------------
DRUG TARGETS -from DRUGBANK and KEGG 
------------------------------------

PREFIX iv: <http://bio2rdf.org/irefindex_vocabulary:>
PREFIX dv: <http://bio2rdf.org/drugbank_vocabulary:>
PREFIX hv: <http://bio2rdf.org/hgnc_vocabulary:>
PREFIX kv: <http://bio2rdf.org/kegg_vocabulary:>
SELECT distinct ?drugid ?geneid 
WHERE
{
?drug <http://bio2rdf.org/openpredict_vocabulary:indication> ?i .
{
  ?d a kv:Drug .
  ?d kv:target ?l .
  ?l kv:link ?t .
  BIND (URI( REPLACE(str(?t),"HSA","hsa")) AS ?target) .
  ?target a kv:Gene .
  ?target kv:x-ncbigene ?ncbi .
  #?target kv:x-uniprot ?ncbi .
  ?d kv:x-drugbank ?drug .
}
UNION
{
  ?drug a dv:Drug .
  ?drug dv:target ?target .
  ?target dv:x-hgnc ?hgnc .
  ?hgnc hv:x-ncbigene ?ncbi .
  #?hgnc hv:uniprot ?ncbi .
}
BIND ( STRAFTER(str(?ncbi),"http://bio2rdf.org/ncbigene:") AS ?geneid)
BIND( STRAFTER(str(?drug), "http://bio2rdf.org/drugbank:") AS ?drugid)
}

------------------------------
DRUG SMILES -- from DRUGBANK
------------------------------

PREFIX dv: <http://bio2rdf.org/drugbank_vocabulary:>

SELECT distinct ?drugid ?smiles
{
 ?d <http://bio2rdf.org/openpredict_vocabulary:indication> ?i .
 ?d dv:calculated-properties ?cp .
 ?cp a dv:SMILES .
 ?cp dv:value ?smiles .
 BIND( STRAFTER(str(?d), "http://bio2rdf.org/drugbank:") AS ?drugid)
}


------------------------------
DRUG SIDE EFFECTS - from SIDER
------------------------------


PREFIX dv: <http://bio2rdf.org/drugbank_vocabulary:>
PREFIX sider: <http://bio2rdf.org/sider_vocabulary:>

SELECT distinct ?drugid ?se
{
?dia <http://bio2rdf.org/sider_vocabulary:drug> ?stitch_flat .
?dia <http://bio2rdf.org/sider_vocabulary:effect> ?se .
{
?stitch_flat <http://bio2rdf.org/sider_vocabulary:x-pubchem.compound> ?pc .
}
UNION{
?stitch_flat <http://bio2rdf.org/sider_vocabulary:stitch-stereo> ?stitch_stereo .
?stitch_stereo <http://bio2rdf.org/sider_vocabulary:x-pubchem.compound> ?pc .
}
?d dv:x-pubchemcompound ?pc  .
BIND(STRAFTER( str(?d), "http://bio2rdf.org/drugbank:") AS ?drugid)
}

------------------------------------------
DISEASE DESCRIPTIONS - from MEDDRA, NDF-RT 
-----------------------------------------

PREFIX sider: <http://bio2rdf.org/sider_vocabulary:>
SELECT distinct ?umlsid ?parent_id #?parent_3
WHERE {
?drug <http://bio2rdf.org/openpredict_vocabulary:indication> ?i .
?a <http://bioportal.bioontology.org/ontologies/umls/cui> ?i .
?a rdfs:subClassOf* ?parent .
FILTER (regex(str(?parent),"^http://bio2rdf.org/ndfrt:") || regex(str(?parent),"^http://bio2rdf.org/meddra:") )
BIND (STRAFTER(str(?i),"http://bio2rdf.org/umls:") AS ?umlsid)
BIND (STRAFTER(str(?parent),"http://bio2rdf.org/") AS ?parent_id)
}


export SPARQL_ENDPOINT=http://localhost:13065/sparql
# query drug target info and downlad in the input folder
curl -H "Accept: text/tab-separated-values" --data-urlencode query@drugbank-drug-target.sparql $SPARQL_ENDPOINT > drugbank-drug-target.tab
# create feature matrix for drug target 
python createTargetFeatureMatrix.py ../data/input/drugbank-drug-target.tab > ../data/features/drugs-targets.txt

# query drug smiles info and downlad in the input folder
curl -H "Accept: text/tab-separated-values" --data-urlencode query@drugbank-drug-smiles.sparql $SPARQL_ENDPOINT > drugbank-drug-smiles.tab
# create feature matrix for drug fingerprint 
python createFingerprintFeatures.py ../data/input/drugbank-drug-smiles.tab > ../data/features/drugs-fingerprint.txt


#disease meddra features
curl -H "Accept: text/tab-separated-values" --data-urlencode query@sider-meddra-terms.sparql $SPARQL_ENDPOINT > sider-meddra-terms.tab
python createMeddraFeatures.py ../data/input/sider-meddra-terms.tab > ../data/features/diseases-meddra.txt


#ensemble all drug and disease feature for given gold standard drug indications (and size of 2*numIndications selected randomly - ( if a negativedisease set is given, negative set is seleceted randomly from diseases wheree no previous indications reported for)

# do 10-fold cross-validation, separate gold standard into train and test by removing 10% of drugs and theirs association
# train on training set and test on test set and report the performance of each fold

 python cv_test.py -g ../data/input/unified-gold-standard-umls.txt -dr ../data/features/drugs-targets.txt ../data/features/drugs-fingerprint.txt ../data/features/drugs-sider-se.txt -di ../data/features/diseases-ndfrt-meddra.txt -o ../data/output/completeset_unified_validation.txt -disjoint 0 -p 2 -m rf
# train with whole gold standard set

# predict new probablitities of unknown relations
 python train_and_test.py -g ../data/input/unified-gold-standard-umls.txt -dr ../data/features/drugs-targets.txt ../data/features/drugs-fingerprint.txt ../data/features/drugs-sider-se.txt -di ../data/features/diseases-ndfrt-meddra.txt -o ../data/predictions/rf_p2_n405.txt -m rf -p 2

