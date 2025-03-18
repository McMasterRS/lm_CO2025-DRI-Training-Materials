---
layout: default
title: APCs Tracking
parent: OpenAlex
nav_order: 2
---

# Tracking Article Processing Charges (APCs) for a given institution

In this notebook, we will query the OpenAlex API to answer the following questions:  

1. **How much are researchers at my institution paying in APCs?**
2. **Which journals/publishers are collecting the most APCs from researchers at my institution?**
3. **How much money are my organization’s researchers saving in discounted APC charges from our transformative/read-publish agreements?**

Most organizations do not have an effective way of tracking the APCs that their researchers pay to publish in open access journals.  By estimating how much money is going to APCs each year, and which publishers are collecting the most APCs, libraries can make more informed decisions around the details of the read-publish agreements they have with various publishers.  

### APC-able Works

Before starting this analysis, it is important to define which types of works are subject to APCs and which are not.   

While a work may include contributions from a number of different institutions, the APC is typically the responsbility of the work's *corresponding author*.  

In addition, open access works published in Gold and Hybrid OA journals are subject to APCs, while those published in Green, Diamond, and Bronze OA journals are not.  

Finally, APCs are not typically charged for editorial content submitted to an open access journal.  

Thus, for the purposes of this notebook, *APC-able works* must have the following characteristics:
- Original articles or reviews
- Published in a Gold or Hybrid OA journal
- Corresponding author is a researcher at our institution.

## Surveying APCs by journal/publisher

### Steps

1. We need to get all the works published and corresponded by researchers at the institution
2. We get the journal/publisher and APC for each publication
3. We sum the APCs (by journal/publisher)

### Input

For inputs, we first need to identify the Research Organization Registry (ROR) ID for our institution. In this example we will use the ROR ID for McMaster University ([https://ror.org/02fa3aq29](https://ror.org/02fa3aq29)). You can search and substitute your own institution's ROR here: [https://ror.org/search](https://ror.org/search).  

Next, we identify the publication year we are interested in analyzing. If the details of your institution's specific tranformative/read-publish agreements change from year to year, you will want to limit your analysis to a single year.  

Finally, becauase editorial content is not typically subject to APCs, we will limit our search to works with the publication types "article" or "review".  


```python
SAVE_CSV = False  # flag to determine whether to save the output as a CSV file 

# input
ror_id = "https://ror.org/02fa3aq29"
publication_year = 2024
publication_types = ["article", "review"]
publication_oa_statuses = ["gold", "hybrid"]
```

### Get OpenAlex ID of the given institution

We only want publications with corresponding authors, who are affiliated with McMaster University. However, OpenAlex currently does not support filtering corresponding institutions by ROR ID, we will need to find out the OpenAlex ID for McMaster using the [`institutions`](https://docs.openalex.org/api-entities/institutions) entity type.  

Our search criteria are as follows:  
- `ror`: ROR ID of the institution, `ror:https://ror.org/02fa3aq2`

Now we need to build an URL for the query from the following parameters:  
- Starting point is the base URL of the OpenAlex API: `https://api.openalex.org/`
- We append the entity type to it: `https://api.openalex.org/institutions`
- All criteria need to go into the query parameter filter that is added after a question mark: `https://api.openalex.org/institutions?filter=`
- To construct the filter value we take the criteria we specified and concatenate them using commas as separators: `https://api.openalex.org/institutions?filter=ror:https://ror.org/02fa3aq29`


```python
import requests

# construct the url using the provided ror id
url = f"https://api.openalex.org/institutions?filter=ror:{ror_id}"

# send a get request to the constructed url
response = requests.get(url)

# parse the response json data
json_data = response.json()

# extract the institution id from the first result
institution_id = json_data["results"][0]["id"]  # https://openalex.org/I98251732
```

### Get all APC-able works published by researchers at the institution

Our search criteria are as follows:  
- `corresponding_institution_ids`: institution affiliated with the corresponding authors of a work (OpenAlex ID), `corresponding_institution_ids:https://openalex.org/I98251732`
- `publication_year`: the year the work was published, `publication_year:2024`
- [`types`](https://docs.openalex.org/api-entities/works/work-object#type): the type of the work, `type:article|review`
- [`oa_status`](https://docs.openalex.org/api-entities/works/work-object#open_access): the OA status of the work, `oa_status:gold|hybrid`

Now we need to build an URL for the query from the following parameters:  
- Starting point is the base URL of the OpenAlex API: `https://api.openalex.org/`
- We append the entity type to it: `https://api.openalex.org/works`
- All criteria need to go into the query parameter filter that is added after a question mark: `https://api.openalex.org/works?filter=`
- To construct the filter value we take the criteria we specified and concatenate them using commas as separators: `https://api.openalex.org/works?filter=corresponding_institution_ids:https://openalex.org/I98251732,publication_year:2024,type:article|review,oa_status:gold|hybrid&page=1&per-page=50`


```python
import numpy as np
import pandas as pd

def get_works_by_institution(institution_id, publication_year, publication_types, page=1, items_per_page=50):
    # construct the api url with the given institution id, publication year, publication types, page number, and items per page
    url = f"https://api.openalex.org/works?filter=corresponding_institution_ids:{institution_id},publication_year:{publication_year},type:{'|'.join(publication_types)},oa_status:{'|'.join(publication_oa_statuses)}&page={page}&per-page={items_per_page}"

    # send a GET request to the api and parse the json response
    response = requests.get(url)
    json_data = response.json()

    # convert the json response to a dataframe
    df_json = pd.DataFrame.from_dict(json_data["results"])

    next_page = True
    if df_json.empty: # check if the dataframe is empty (i.e., no more pages available)
        next_page = False

    # if there are more pages, recursively fetch the next page
    if next_page:
        df_json_next_page = get_works_by_institution(institution_id, publication_year, publication_types, page=page+1, items_per_page=items_per_page)
        df_json = pd.concat([df_json, df_json_next_page])

    return df_json
```


```python
df_works = get_works_by_institution(institution_id, publication_year, publication_types)
if SAVE_CSV:
    df_works.to_csv(f"institution_works_{publication_year}.csv", index=True)

df_works
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>doi</th>
      <th>title</th>
      <th>display_name</th>
      <th>publication_year</th>
      <th>publication_date</th>
      <th>ids</th>
      <th>language</th>
      <th>primary_location</th>
      <th>type</th>
      <th>...</th>
      <th>versions</th>
      <th>referenced_works_count</th>
      <th>referenced_works</th>
      <th>related_works</th>
      <th>abstract_inverted_index</th>
      <th>abstract_inverted_index_v3</th>
      <th>cited_by_api_url</th>
      <th>counts_by_year</th>
      <th>updated_date</th>
      <th>created_date</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>https://openalex.org/W4391340020</td>
      <td>https://doi.org/10.1002/adfm.202314520</td>
      <td>Borophene Based 3D Extrusion Printed Nanocompo...</td>
      <td>Borophene Based 3D Extrusion Printed Nanocompo...</td>
      <td>2024</td>
      <td>2024-01-30</td>
      <td>{'openalex': 'https://openalex.org/W4391340020...</td>
      <td>en</td>
      <td>{'is_oa': True, 'landing_page_url': 'https://d...</td>
      <td>article</td>
      <td>...</td>
      <td>[]</td>
      <td>72</td>
      <td>[https://openalex.org/W1963693172, https://ope...</td>
      <td>[https://openalex.org/W4313334364, https://ope...</td>
      <td>{'Abstract': [0], 'Herein,': [1], 'a': [2, 10,...</td>
      <td>None</td>
      <td>https://api.openalex.org/works?filter=cites:W4...</td>
      <td>[{'year': 2025, 'cited_by_count': 7}, {'year':...</td>
      <td>2025-03-17T19:08:42.310588</td>
      <td>2024-01-31</td>
    </tr>
    <tr>
      <th>1</th>
      <td>https://openalex.org/W4391115198</td>
      <td>https://doi.org/10.1002/anie.202318665</td>
      <td>Development of Better Aptamers: Structured Lib...</td>
      <td>Development of Better Aptamers: Structured Lib...</td>
      <td>2024</td>
      <td>2024-01-23</td>
      <td>{'openalex': 'https://openalex.org/W4391115198...</td>
      <td>en</td>
      <td>{'is_oa': True, 'landing_page_url': 'https://d...</td>
      <td>review</td>
      <td>...</td>
      <td>[]</td>
      <td>204</td>
      <td>[https://openalex.org/W1509198841, https://ope...</td>
      <td>[https://openalex.org/W4313237059, https://ope...</td>
      <td>{'Abstract': [0], 'Systematic': [1], 'evolutio...</td>
      <td>None</td>
      <td>https://api.openalex.org/works?filter=cites:W4...</td>
      <td>[{'year': 2025, 'cited_by_count': 11}, {'year'...</td>
      <td>2025-03-14T02:50:47.690863</td>
      <td>2024-01-23</td>
    </tr>
    <tr>
      <th>2</th>
      <td>https://openalex.org/W4392384326</td>
      <td>https://doi.org/10.1016/s2666-7568(24)00007-2</td>
      <td>Prevalence of multimorbidity and polypharmacy ...</td>
      <td>Prevalence of multimorbidity and polypharmacy ...</td>
      <td>2024</td>
      <td>2024-03-04</td>
      <td>{'openalex': 'https://openalex.org/W4392384326...</td>
      <td>en</td>
      <td>{'is_oa': True, 'landing_page_url': 'https://d...</td>
      <td>review</td>
      <td>...</td>
      <td>[]</td>
      <td>127</td>
      <td>[https://openalex.org/W1056226603, https://ope...</td>
      <td>[https://openalex.org/W3196849760, https://ope...</td>
      <td>{'Multimorbidity': [0], '(multiple': [1, 5], '...</td>
      <td>None</td>
      <td>https://api.openalex.org/works?filter=cites:W4...</td>
      <td>[{'year': 2025, 'cited_by_count': 8}, {'year':...</td>
      <td>2025-03-12T16:35:56.335359</td>
      <td>2024-03-05</td>
    </tr>
    <tr>
      <th>3</th>
      <td>https://openalex.org/W4391180419</td>
      <td>https://doi.org/10.1016/j.molliq.2024.124105</td>
      <td>Rapid and effective antibiotics elimination fr...</td>
      <td>Rapid and effective antibiotics elimination fr...</td>
      <td>2024</td>
      <td>2024-01-24</td>
      <td>{'openalex': 'https://openalex.org/W4391180419...</td>
      <td>en</td>
      <td>{'is_oa': True, 'landing_page_url': 'https://d...</td>
      <td>article</td>
      <td>...</td>
      <td>[]</td>
      <td>62</td>
      <td>[https://openalex.org/W1988636074, https://ope...</td>
      <td>[https://openalex.org/W888754083, https://open...</td>
      <td>{'Tetracycline': [0], '(TC)': [1], 'and': [2, ...</td>
      <td>None</td>
      <td>https://api.openalex.org/works?filter=cites:W4...</td>
      <td>[{'year': 2025, 'cited_by_count': 7}, {'year':...</td>
      <td>2025-03-11T00:28:29.517999</td>
      <td>2024-01-25</td>
    </tr>
    <tr>
      <th>4</th>
      <td>https://openalex.org/W4390610871</td>
      <td>https://doi.org/10.1016/j.snb.2024.135282</td>
      <td>Interleukin-6 electrochemical sensor using pol...</td>
      <td>Interleukin-6 electrochemical sensor using pol...</td>
      <td>2024</td>
      <td>2024-01-05</td>
      <td>{'openalex': 'https://openalex.org/W4390610871...</td>
      <td>en</td>
      <td>{'is_oa': True, 'landing_page_url': 'https://d...</td>
      <td>article</td>
      <td>...</td>
      <td>[]</td>
      <td>67</td>
      <td>[https://openalex.org/W1274997689, https://ope...</td>
      <td>[https://openalex.org/W2385756659, https://ope...</td>
      <td>{'Interleukin-6': [0], '(IL-6)': [1], 'is': [2...</td>
      <td>None</td>
      <td>https://api.openalex.org/works?filter=cites:W4...</td>
      <td>[{'year': 2025, 'cited_by_count': 2}, {'year':...</td>
      <td>2025-03-04T15:16:23.540238</td>
      <td>2024-01-07</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>47</th>
      <td>https://openalex.org/W4400490490</td>
      <td>https://doi.org/10.1371/journal.pone.0306075</td>
      <td>Receipt of Opioid Agonist Treatment in provinc...</td>
      <td>Receipt of Opioid Agonist Treatment in provinc...</td>
      <td>2024</td>
      <td>2024-07-10</td>
      <td>{'openalex': 'https://openalex.org/W4400490490...</td>
      <td>en</td>
      <td>{'is_oa': True, 'landing_page_url': 'https://d...</td>
      <td>article</td>
      <td>...</td>
      <td>[]</td>
      <td>58</td>
      <td>[https://openalex.org/W1533521761, https://ope...</td>
      <td>[https://openalex.org/W3177241792, https://ope...</td>
      <td>{'In': [0], 'many': [1], 'jurisdictions,': [2]...</td>
      <td>None</td>
      <td>https://api.openalex.org/works?filter=cites:W4...</td>
      <td>[]</td>
      <td>2025-03-18T07:35:19.142891</td>
      <td>2024-07-11</td>
    </tr>
    <tr>
      <th>48</th>
      <td>https://openalex.org/W4400515928</td>
      <td>https://doi.org/10.1111/sjop.13053</td>
      <td>Sociability across Eastern–Western cultures: I...</td>
      <td>Sociability across Eastern–Western cultures: I...</td>
      <td>2024</td>
      <td>2024-07-09</td>
      <td>{'openalex': 'https://openalex.org/W4400515928...</td>
      <td>en</td>
      <td>{'is_oa': True, 'landing_page_url': 'https://d...</td>
      <td>article</td>
      <td>...</td>
      <td>[]</td>
      <td>67</td>
      <td>[https://openalex.org/W1492236751, https://ope...</td>
      <td>[https://openalex.org/W68679956, https://opena...</td>
      <td>{'In': [0], 'this': [1, 59], 'study,': [2], 'w...</td>
      <td>None</td>
      <td>https://api.openalex.org/works?filter=cites:W4...</td>
      <td>[]</td>
      <td>2025-03-18T07:43:08.509042</td>
      <td>2024-07-11</td>
    </tr>
    <tr>
      <th>49</th>
      <td>https://openalex.org/W4401060634</td>
      <td>https://doi.org/10.1016/j.vaccine.2024.07.023</td>
      <td>The role of influenza Hemagglutination-Inhibit...</td>
      <td>The role of influenza Hemagglutination-Inhibit...</td>
      <td>2024</td>
      <td>2024-07-28</td>
      <td>{'openalex': 'https://openalex.org/W4401060634...</td>
      <td>en</td>
      <td>{'is_oa': True, 'landing_page_url': 'https://d...</td>
      <td>article</td>
      <td>...</td>
      <td>[]</td>
      <td>29</td>
      <td>[https://openalex.org/W1970848116, https://ope...</td>
      <td>[https://openalex.org/W3023742952, https://ope...</td>
      <td>{'Influenza': [0], 'vaccination': [1, 17], 'ma...</td>
      <td>None</td>
      <td>https://api.openalex.org/works?filter=cites:W4...</td>
      <td>[]</td>
      <td>2025-03-18T08:45:08.115726</td>
      <td>2024-07-31</td>
    </tr>
    <tr>
      <th>0</th>
      <td>https://openalex.org/W4403306060</td>
      <td>https://doi.org/10.3390/ijms252010897</td>
      <td>Myeloid GSK3α Deficiency Reduces Lesional Infl...</td>
      <td>Myeloid GSK3α Deficiency Reduces Lesional Infl...</td>
      <td>2024</td>
      <td>2024-10-10</td>
      <td>{'openalex': 'https://openalex.org/W4403306060...</td>
      <td>en</td>
      <td>{'is_oa': True, 'landing_page_url': 'https://d...</td>
      <td>article</td>
      <td>...</td>
      <td>[]</td>
      <td>27</td>
      <td>[https://openalex.org/W1487848458, https://ope...</td>
      <td>[https://openalex.org/W73242545, https://opena...</td>
      <td>{'The': [0, 50, 100], 'molecular': [1], 'mecha...</td>
      <td>None</td>
      <td>https://api.openalex.org/works?filter=cites:W4...</td>
      <td>[]</td>
      <td>2025-03-18T12:07:28.678331</td>
      <td>2024-10-11</td>
    </tr>
    <tr>
      <th>1</th>
      <td>https://openalex.org/W4405314055</td>
      <td>https://doi.org/10.1002/ksa.12556</td>
      <td>Anterior cruciate ligament reconstruction with...</td>
      <td>Anterior cruciate ligament reconstruction with...</td>
      <td>2024</td>
      <td>2024-12-12</td>
      <td>{'openalex': 'https://openalex.org/W4405314055...</td>
      <td>en</td>
      <td>{'is_oa': True, 'landing_page_url': 'https://d...</td>
      <td>review</td>
      <td>...</td>
      <td>[]</td>
      <td>50</td>
      <td>[https://openalex.org/W1958440778, https://ope...</td>
      <td>[https://openalex.org/W4401483477, https://ope...</td>
      <td>{'Abstract': [0], 'Purpose': [1], 'This': [2],...</td>
      <td>None</td>
      <td>https://api.openalex.org/works?filter=cites:W4...</td>
      <td>[]</td>
      <td>2025-03-18T12:34:56.261172</td>
      <td>2024-12-13</td>
    </tr>
  </tbody>
</table>
<p>952 rows × 51 columns</p>
</div>



### Get Journals/Publishers and APCs in USD

In a `work` entity object, there is information about the journal (`primary_location`) and the journal's APC list price ([`apc_list`](https://docs.openalex.org/api-entities/works/work-object#apc_list)).  

is derived from the [Directory of Open Access Journals (DOAJ)](https://doaj.org/), which compiles APC data currently available on publishers' websites.  

It should be noted that not all publishers list APC prices on their websites, meaning that not all works will have an `apc_list` price in OpenAlex.  In these cases, we will infer the APC price based on the mean APC prices of those works for which `apc_list` data is available.  

In addition, even when APC price is included on a publishers' website, there is no guarantee that this is the final APC price our authors paid for publication.  

For these reasons, results of this notebook must be understood as best available estimates.  


```python
# extract 'value_usd' from 'apc_list' if it is a dictionary (i.e. 'apc_list' exists in the work record); otherwise, set to null
df_works["apc_list_usd"] = df_works["apc_list"].apply(lambda apc_list: apc_list["value_usd"] if isinstance(apc_list, dict) else np.nan)

# extract 'id' and 'name' from 'source' within 'primary_location' if 'source' exists; otherwise, set to null
df_works["source_id"] = df_works["primary_location"].apply(lambda location: location["source"]["id"] if location["source"] else np.nan)
df_works["source_name"] = df_works["primary_location"].apply(lambda location: location["source"]["display_name"] if location["source"] else np.nan)

# extract 'host_organization' from 'source' within 'primary_location' if 'source' exists; otherwise, set to null
df_works["source_host_organization"] = df_works["primary_location"].apply(lambda location: location["source"]["host_organization"] if location["source"] else np.nan)

# extract 'issn' and 'issn_l' from 'source' within 'primary_location' if 'source' exists; otherwise, set to null
df_works["source_issn"] = df_works["primary_location"].apply(lambda location: location["source"]["issn"] if location["source"] else np.nan)
df_works["source_issn_l"] = df_works["primary_location"].apply(lambda location: location["source"]["issn_l"] if location["source"] else np.nan)
```


```python
# calculate the average apc where 'apc_list_usd' is not null
apc_mean = df_works[df_works["apc_list_usd"].notnull()]["apc_list_usd"].mean()

# fill null values in 'apc_list_usd' with the calculated average
df_works["apc_list_usd"] = df_works["apc_list_usd"].fillna(apc_mean)

# fill null values in 'source_id', 'source_name', 'source_issn' and 'source_issn_l'
df_works["source_id"] = df_works["source_id"].fillna("unknown source")
df_works["source_name"] = df_works["source_name"].fillna("unknown source")
df_works["source_host_organization"] = df_works["source_host_organization"].fillna("unknown source")
df_works["source_issn"] = df_works["source_issn"].fillna("unknown source")
df_works["source_issn_l"] = df_works["source_issn_l"].fillna("unknown source")
```

### Get Publisher Display Name (Optional)

OpenAlex identifies publishers with a unique identfier called an OpenAlex ID. The following code translates this OpenAlex ID to the publisher's display name for easier analysis.  


```python
import re

CHUNK_SIZE = 5

def get_source_host_organization_display_name(publisher_ids):
    def get_source_host_organization_publisher_display_name(publisher_ids):
        def get_source_host_organization_publisher_display_name_by_chunk(publisher_ids_chunk):
            # construct the api url using the chunk of publisher ids
            url = f"https://api.openalex.org/publishers?filter=ids.openalex:{'|'.join(publisher_ids_chunk)}"

            # send a GET request to the api and parse the json response
            response = requests.get(url)
            json_data = response.json()

            # convert the json response to a dataframe and return the relevant columns
            df_json = pd.DataFrame.from_dict(json_data["results"])
            return df_json[["id", "display_name"]]
        
        # check if the length of 'publisher_ids' is less than 1
        if len(publisher_ids) < 1:
            # if true, return an empty dataframe
            return pd.DataFrame()

        # split the publisher ids into chunks and apply the function to each chunk
        chunks = np.array_split(publisher_ids, np.ceil(len(publisher_ids) / CHUNK_SIZE))
        df_chunks = pd.DataFrame({"chunk": chunks})
        return pd.concat(df_chunks["chunk"].apply(get_source_host_organization_publisher_display_name_by_chunk).tolist())

    def get_source_host_organization_institution_display_name(institution_ids):
        def get_source_host_organization_institution_display_name_by_chunk(institution_ids_chunk):
            # construct the api url using the chunk of institution ids
            url = f"https://api.openalex.org/institutions?filter=id:{'|'.join(institution_ids_chunk)}"

            # send a GET request to the api and parse the json response
            response = requests.get(url)
            json_data = response.json()

            # convert the json response to a dataframe and return the relevant columns
            df_json = pd.DataFrame.from_dict(json_data["results"])
            return df_json[["id", "display_name"]]

        # check if the length of 'institution_ids' is less than 1
        if len(institution_ids) < 1:
            # if true, return an empty dataframe
            return pd.DataFrame()

        # split the institution ids into chunks and apply the function to each chunk
        chunks = np.array_split(institution_ids, np.ceil(len(institution_ids) / CHUNK_SIZE))
        df_chunks = pd.DataFrame({"chunk": chunks})
        return pd.concat(df_chunks["chunk"].apply(get_source_host_organization_institution_display_name_by_chunk).tolist())

     # filter the publisher ids to get only publisher urls
    publishers = list(filter(lambda s: re.search(r"https:\/\/openalex\.org\/P", s), publisher_ids))
    # filter the institution ids to get only institution urls
    institutions = list(filter(lambda s: re.search(r"https:\/\/openalex\.org\/I", s), publisher_ids))

    # create a dataframe with a default entry for unknown source
    df_lookup = pd.DataFrame.from_dict({"id": ["unknown source"], "display_name": ["unknown source"]})
    # concatenate the dataframes with publisher and institution display names
    df_lookup = pd.concat([df_lookup, get_source_host_organization_publisher_display_name(publishers), get_source_host_organization_institution_display_name(institutions)], ignore_index=True)
    return df_lookup
```


```python
# get the display names for unique source_host_organization ids in df_works
df_lookup = get_source_host_organization_display_name(df_works["source_host_organization"].unique())
if SAVE_CSV:
    df_lookup.to_csv(f"source_host_organization_lookup.csv", index=True)

df_lookup
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>display_name</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>unknown source</td>
      <td>unknown source</td>
    </tr>
    <tr>
      <th>1</th>
      <td>https://openalex.org/P4310320990</td>
      <td>Elsevier BV</td>
    </tr>
    <tr>
      <th>2</th>
      <td>https://openalex.org/P4310320595</td>
      <td>Wiley</td>
    </tr>
    <tr>
      <th>3</th>
      <td>https://openalex.org/P4310319908</td>
      <td>Nature Portfolio</td>
    </tr>
    <tr>
      <th>4</th>
      <td>https://openalex.org/P4310320556</td>
      <td>Royal Society of Chemistry</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>57</th>
      <td>https://openalex.org/P4310312949</td>
      <td>Surveillance Studies Network</td>
    </tr>
    <tr>
      <th>58</th>
      <td>https://openalex.org/P4310311647</td>
      <td>University of Oxford</td>
    </tr>
    <tr>
      <th>59</th>
      <td>https://openalex.org/P4310319847</td>
      <td>Routledge</td>
    </tr>
    <tr>
      <th>60</th>
      <td>https://openalex.org/P4310319955</td>
      <td>Academic Press</td>
    </tr>
    <tr>
      <th>61</th>
      <td>https://openalex.org/P4324113769</td>
      <td>Iter Press</td>
    </tr>
  </tbody>
</table>
<p>62 rows × 2 columns</p>
</div>




```python
# update the 'source_host_organization' with the corresponding display names
df_works["source_host_organization"] = df_works["source_host_organization"].apply(lambda publisher: df_lookup[df_lookup["id"] == publisher]["display_name"].squeeze())  
```

### Aggregate APCs Data

Here, we build a dataframe containing the number of APC-able works and the estiamted total APC cost for each journal.


```python
# group the dataframe by 'source_id' and 'source_issn_l'
# and aggregate 'source_issn' by taking the maximum value (in this case the common issn list of strings)
# and aggregate 'source_host_organization' by taking the maximum value (in this case the common string name of the source's host organization)
# and aggregate 'source_name' by taking the maximum value (in this case the common string name of the source)
# and 'id' by counting
# and 'apc_list_usd' by summing
df_apc = df_works.groupby(["source_id", "source_issn_l"]).agg({"source_issn": "max", "source_name": "max", "source_host_organization": "max", "id": "count", "apc_list_usd": "sum"})
# rename the 'id' column to 'num_publications' and 'apc_list_usd' column to 'apc_usd'
df_apc.rename(columns={"id": "num_publications", "apc_list_usd": "apc_usd"}, inplace=True)
if SAVE_CSV:
    df_apc.to_csv(f"apc_usd_by_source.csv", index=True)

df_apc
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th></th>
      <th>source_issn</th>
      <th>source_name</th>
      <th>source_host_organization</th>
      <th>num_publications</th>
      <th>apc_usd</th>
    </tr>
    <tr>
      <th>source_id</th>
      <th>source_issn_l</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>https://openalex.org/S100014455</th>
      <th>1756-0500</th>
      <td>[1756-0500]</td>
      <td>BMC Research Notes</td>
      <td>BioMed Central</td>
      <td>1</td>
      <td>1361.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S100662246</th>
      <th>1748-2623</th>
      <td>[1748-2623, 1748-2631]</td>
      <td>International Journal of Qualitative Studies o...</td>
      <td>Taylor &amp; Francis</td>
      <td>1</td>
      <td>1790.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S100695177</th>
      <th>0004-6256</th>
      <td>[0004-6256, 1538-3881]</td>
      <td>The Astronomical Journal</td>
      <td>Institute of Physics</td>
      <td>1</td>
      <td>4499.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S10134376</th>
      <th>2071-1050</th>
      <td>[2071-1050]</td>
      <td>Sustainability</td>
      <td>Multidisciplinary Digital Publishing Institute</td>
      <td>2</td>
      <td>4764.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S101949793</th>
      <th>1424-8220</th>
      <td>[1424-8220]</td>
      <td>Sensors</td>
      <td>Multidisciplinary Digital Publishing Institute</td>
      <td>5</td>
      <td>12990.0</td>
    </tr>
    <tr>
      <th>...</th>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>https://openalex.org/S98651283</th>
      <th>0022-4227</th>
      <td>[0022-4227, 1467-9809]</td>
      <td>Journal of Religious History</td>
      <td>Wiley</td>
      <td>1</td>
      <td>2630.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S99352657</th>
      <th>0935-9648</th>
      <td>[0935-9648, 1521-4095]</td>
      <td>Advanced Materials</td>
      <td>unknown source</td>
      <td>2</td>
      <td>10500.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S99498898</th>
      <th>1567-5394</th>
      <td>[1567-5394, 1878-562X]</td>
      <td>Bioelectrochemistry</td>
      <td>Elsevier BV</td>
      <td>1</td>
      <td>3370.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S99546260</th>
      <th>1836-9561</th>
      <td>[1836-9561, 1836-9553]</td>
      <td>Journal of physiotherapy</td>
      <td>Elsevier BV</td>
      <td>1</td>
      <td>3450.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S99985186</th>
      <th>1360-8592</th>
      <td>[1360-8592, 1532-9283]</td>
      <td>Journal of Bodywork and Movement Therapies</td>
      <td>Elsevier BV</td>
      <td>1</td>
      <td>2670.0</td>
    </tr>
  </tbody>
</table>
<p>576 rows × 5 columns</p>
</div>



### Estimating the total (non-discounted) APC spend


```python
total_apc = df_apc["apc_usd"].sum()
print(f"Estimated total (non-discounted) APC cost in {publication_year}: USD {round(total_apc, 2)}.")
```

    Estimated total (non-discounted) APC cost in 2024: USD 2754479.12.


## Calculating Discounted APCs

### Steps

1. We load the given list of read-publish agreement discounts
2. We check if publishers are included in the list by ISSN
3. We calculate the APCs paid with the list of read-publish agreement discounts and the APC listed prior

### Input

Assume we have list of read-publish agreement discounts in CSV format, `discount-list.csv`. In the file, it includes the following necessary attributes,  
- `issn`: ISSN of the publisher
- `discount`: value of the discount, either a number or a percentage
- `is_flatrate`: flag indicating whether the discount is a flat rate discount or a percentage discount

You can download a [template `discount-list.csv`](https://github.com/McMasterRS/lm_CO2025-deliverables/blob/main/openalex/discount-list-sample.csv) here and update it with the details of your institutions own APC discounts.  


```python
# input
df_discount = pd.read_csv("discount-list.csv")
```


```python
import typing

def get_discount(issn: typing.List[str] | str, apc: float) -> float:
    # check if issn is a string, if so, convert it to a list
    if isinstance(issn, str):
        issn = [issn]

    # filter the discount dataframe to get rows where 'issn' is in the provided issn list
    discount_rows = df_discount[df_discount["issn"].isin(issn)]
    
    # if no discount rows are found, return the original apc
    if discount_rows.empty:
        return apc

    # get the first row from the filtered discount rows
    discount_row = discount_rows.iloc[0]
    
    # if the discount is a flat rate, subtract the discount from the apc
    if discount_row["is_flatrate"]:
        return apc - discount_row["discount"]
    else:
        # if the discount is a percentage, apply the discount to the apc
        return apc * (1 - discount_row["discount"])
```

### Apply Discounts to Aggregated APC Data

Here, we apply the APC discounts to the aggregated APC data. This produces a dataframe and `.csv` file that includes the number of APC-able publications and the discounted APC cost for each journal.  


```python
# apply the get_discount function to each row of the dataframe to calculate the discounted apc and store it in a new column 'discounted_apc_usd'
df_apc["discounted_apc_usd"] = df_apc.apply(lambda x: get_discount(issn=x["source_issn"], apc=x["apc_usd"]), axis=1)
if SAVE_CSV:
    df_apc.to_csv(f"apc_usd_with_discounts_by_source.csv", index=True)

df_apc
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th></th>
      <th>source_issn</th>
      <th>source_name</th>
      <th>source_host_organization</th>
      <th>num_publications</th>
      <th>apc_usd</th>
      <th>discounted_apc_usd</th>
    </tr>
    <tr>
      <th>source_id</th>
      <th>source_issn_l</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>https://openalex.org/S100014455</th>
      <th>1756-0500</th>
      <td>[1756-0500]</td>
      <td>BMC Research Notes</td>
      <td>BioMed Central</td>
      <td>1</td>
      <td>1361.0</td>
      <td>1361.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S100662246</th>
      <th>1748-2623</th>
      <td>[1748-2623, 1748-2631]</td>
      <td>International Journal of Qualitative Studies o...</td>
      <td>Taylor &amp; Francis</td>
      <td>1</td>
      <td>1790.0</td>
      <td>1790.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S100695177</th>
      <th>0004-6256</th>
      <td>[0004-6256, 1538-3881]</td>
      <td>The Astronomical Journal</td>
      <td>Institute of Physics</td>
      <td>1</td>
      <td>4499.0</td>
      <td>4499.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S10134376</th>
      <th>2071-1050</th>
      <td>[2071-1050]</td>
      <td>Sustainability</td>
      <td>Multidisciplinary Digital Publishing Institute</td>
      <td>2</td>
      <td>4764.0</td>
      <td>4764.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S101949793</th>
      <th>1424-8220</th>
      <td>[1424-8220]</td>
      <td>Sensors</td>
      <td>Multidisciplinary Digital Publishing Institute</td>
      <td>5</td>
      <td>12990.0</td>
      <td>12990.0</td>
    </tr>
    <tr>
      <th>...</th>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>https://openalex.org/S98651283</th>
      <th>0022-4227</th>
      <td>[0022-4227, 1467-9809]</td>
      <td>Journal of Religious History</td>
      <td>Wiley</td>
      <td>1</td>
      <td>2630.0</td>
      <td>2630.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S99352657</th>
      <th>0935-9648</th>
      <td>[0935-9648, 1521-4095]</td>
      <td>Advanced Materials</td>
      <td>unknown source</td>
      <td>2</td>
      <td>10500.0</td>
      <td>10500.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S99498898</th>
      <th>1567-5394</th>
      <td>[1567-5394, 1878-562X]</td>
      <td>Bioelectrochemistry</td>
      <td>Elsevier BV</td>
      <td>1</td>
      <td>3370.0</td>
      <td>3370.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S99546260</th>
      <th>1836-9561</th>
      <td>[1836-9561, 1836-9553]</td>
      <td>Journal of physiotherapy</td>
      <td>Elsevier BV</td>
      <td>1</td>
      <td>3450.0</td>
      <td>3450.0</td>
    </tr>
    <tr>
      <th>https://openalex.org/S99985186</th>
      <th>1360-8592</th>
      <td>[1360-8592, 1532-9283]</td>
      <td>Journal of Bodywork and Movement Therapies</td>
      <td>Elsevier BV</td>
      <td>1</td>
      <td>2670.0</td>
      <td>2670.0</td>
    </tr>
  </tbody>
</table>
<p>576 rows × 6 columns</p>
</div>



### Estimating the total discounted APC cost


```python
total_apc_discount = df_apc["discounted_apc_usd"].sum()
print(f"Estimated APC cost (including discounts) for {publication_year}: USD {round(total_apc_discount, 2)}.")
```

    Estimated APC cost (including discounts) for 2024: USD 2584462.85.


### Estimating the total APC savings of our institution's read-publish agreements


```python
print(f"Estimated APC saving in {publication_year}: USD {round(total_apc - total_apc_discount, 2)}.")
```

    Estimated APC saving in 2024: USD 170016.27.

