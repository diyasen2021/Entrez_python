# Automate PubMed Searches with Python: A Step-by-Step Guide for Bioinformatics Beginners

If you've ever wanted to retrieve research articles from PubMed in an automated way, this Python script will do just that. We'll be using the Entrez module from Biopython, for fetching articles from PubMed. 
Let’s dive in and transform what could be hours of manual work into a few seconds of automated magic. 🎩✨

## Why Automate PubMed Searches?
Imagine being able to query PubMed with a term like "antibiotic resistance" and instantly get a list of articles with titles and abstracts that go straight into a CSV file. Here's what you'll achieve by the end of this tutorial:

Search PubMed for any term you want.
Extract titles and abstracts for matching articles.
Save your results as a CSV file for easy analysis.

## What You’ll Need
Before we jump into the code, make sure you have:

1. Python installed on your computer.
2. The Biopython library, which gives us access to NCBI’s Entrez API.
3. A curious mind! 🧠


## 1. Setting up your python environment

The script needs the Biopython Entrez module for fetching data from NCBI, pandas for data handling, and urllib.error for managing HTTP-related exceptions.

Every PubMed query requires an email address to use the Entrez API. Replace 'youremail@example.com' with your actual email address.

```
import time
from Bio import Entrez
import pandas as pd
import urllib.error  # For handling HTTP errors

# Provide your email address to the Entrez system
Entrez.email = "youremail@example.com"
```

## 2. Searching PubMed for Articles

We’ll start with a simple function to search PubMed for articles matching your query (e.g., "CRISPR gene editing"). This function retrieves a list of unique article IDs for further processing.

The Entrez esearch function is used here to perform the search.

```
def search(query, retmax=10000):
    """Fetch all IDs matching the query in a single search."""
    handle = Entrez.esearch(db='pubmed',
                            sort='relevance',
                            retmax=retmax,  # Fetch maximum number of results
                            retmode='xml',
                            term=query)
    results = Entrez.read(handle)
    return results
```

## 3. Fetching Article Details

Now comes the exciting part: retrieving titles and abstracts. We’ll use another function to take the article IDs from the previous step and fetch their details.

```
def fetch_details(id_list):
    """Fetch details of articles using a list of PubMed IDs."""
    ids = ','.join(id_list)
    handle = Entrez.efetch(db='pubmed',
                           retmode='xml',
                           id=ids)
    results = Entrez.read(handle)
    return results
```

## 4. Executing the code

Here’s how the entire workflow looks:

1. Search PubMed using your query.
2. Split the results into manageable batches (to avoid overwhelming PubMed’s servers).
3. Fetch article details in each batch. This batch approach ensures efficient data retrieval while respecting PubMed's usage limitations, enabling the processing of large datasets in a structured and manageable way.
4. Save the titles and abstracts to a CSV file.

The main code block:

```
if __name__ == '__main__':
    term = "CRISPR gene editing"  # Your search term

    # Fetch all PubMed IDs for the query
    initial_results = esearch(term)
    total_count = int(initial_results['Count'])
    id_list = initial_results['IdList']
    print(f"Total number of articles: {total_count}")

    # Variables for batch processing
    title_list = []
    abstract_list = []
    batch_size = 50
    sleep_time = 10

    # Fetch articles in batches 
    for start in range(0, len(id_list), batch_size):
        batch_ids = id_list[start:start + batch_size]

        time.sleep(sleep_time)  # Be polite to NCBI servers
        papers = efetch(batch_ids)

        for paper in papers['PubmedArticle']:
            try:
                title = paper['MedlineCitation']['Article']['ArticleTitle']
                title_list.append(title)
            except:
                title_list.append("NA")
            try:
                abstract = paper['MedlineCitation']['Article']['Abstract']['AbstractText'][0]
                abstract_list.append(abstract)
            except:
                abstract_list.append("NA")

    # Save results to a CSV file
    df = pd.DataFrame(list(zip(title_list, abstract_list)), columns=['Title', 'Abstract'])
    df.to_csv('pubmed_articles.csv', index=False)
    print(f"Articles saved to pubmed_articles.csv")
```

## 5. Putting it all together

```
import time
from Bio import Entrez
import pandas as pd
import urllib.error

# Provide your email address to the Entrez system
Entrez.email = "youremail@example.com"

def esearch(query, retmax=10000):
    """Fetch all IDs matching the query in a single search."""
    handle = Entrez.esearch(db='pubmed',
                            sort='relevance',
                            retmax=retmax,  # Fetch maximum number of results
                            retmode='xml',
                            term=query)
    results = Entrez.read(handle)
    return results

def efetch(id_list):
    """Fetch details of articles using a list of PubMed IDs."""
    print(f"Fetching {len(id_list)} IDs")
    ids = ','.join(id_list)
    handle = Entrez.efetch(db='pubmed',
                           retmode='xml',
                           id=ids)
    results = Entrez.read(handle)
    return results

if __name__ == '__main__':
    term = "CRISPR gene editing"

    # Initialize variables
    title_list, abstract_list = [], []
    batch_size = 50  # Fetch articles in batches of XX
    sleep_time = 10  # Sleep for 10 seconds between requests
    max_retries = 5  # Maximum number of retries for a failed batch
    
    # Single search to get all PubMed IDs for the query
    initial_results = esearch(term)
    total_count = int(initial_results['Count'])
    id_list = initial_results['IdList']
    print(f"Total number of articles: {total_count}")

    # Process the list of IDs in batches using efetch
    for start in range(0, len(id_list), batch_size):
        batch_ids = id_list[start:start+batch_size]  # Get a chunk of IDs

        retries = 0
        while retries < max_retries:
            try:
                # Sleep before making the efetch request to avoid rate limits
                time.sleep(sleep_time)

                # Fetch article details for the current batch
                papers = efetch(batch_ids)

                # Process the fetched papers
                for i, paper in enumerate(papers['PubmedArticle']):
                    try:
                        title = paper['MedlineCitation']['Article']['ArticleTitle']
                        title_list.append(title)
                    except:
                        title_list.append("NA")
                    try:
                        abstract = paper['MedlineCitation']['Article']['Abstract']['AbstractText'][0]
                        abstract_list.append(abstract)
                    except:
                        abstract_list.append("NA")

                # Break out of the retry loop if the batch is successful
                break

            except urllib.error.HTTPError as e:
                # Handle HTTP error and retry the batch
                retries += 1
                print(f"HTTP error occurred: {e}, retrying... ({retries}/{max_retries})")
                if retries == max_retries:
                    print(f"Max retries reached for batch {batch_ids}. Skipping this batch.")

    # Create a DataFrame from the collected titles and abstracts
    data = list(zip(title_list, abstract_list))
    df = pd.DataFrame(data, columns=['Title', 'Abstract'])
    df.to_csv('pubmed_articles.csv', index=False)

    # Print the DataFrame to check the result
    print(df)
```

## 6. Ready to Automate Your Research?
This script is just the tip of the iceberg. Once you master the basics, you can build more advanced tools, like filtering articles by publication date or downloading metadata for specific journals.

Automation is a superpower. Go ahead—give this a try, and let me know what you build next! 🚀