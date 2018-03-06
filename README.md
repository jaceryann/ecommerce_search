## Introduction

The goal for this project is to take a specified list of search terms, use each one in a basic search on [Amazon](https://www.amazon.com/), and pull key data (product name, price, rating) from the resulting page. This GitHub Page will walk through each of the steps required in detail, addressing ways to capture the key data when web design differences may apply.

## GET HTML

The first step is to GET the web page HTML by using the API of a PoolManager (using `urllib3`). As recommended by the `urllib3` [documentation](https://urllib3.readthedocs.io/en/latest/user-guide.html#ssl), SSL certificate verification should be used when making HTTPS requests. Mozilla's root certificate bundle is available via the `certifi` package.

```python
import certifi
import urllib3

http = urllib3.PoolManager(cert_reqs='CERT_REQUIRED', 
                           ca_certs=certifi.where())
```

Starting at the home page for Amazon, manually enter one of the specified searches in order to see how the URL changes in order to direct to that search request. In this example, the search 'python books' leads to the following URL:

> 'https://www.amazon.com/s/ref=nb_sb_noss_1?url=search-alias%3Daps&field-keywords=python+books'

This indicates that the base URL for the search results page is everything up to `&field-keywords=` followed by the search terms separated by a `+` sign. A couple of tests typing different search terms directly into the URL in this fashion will verify this assumption. 

The next step, then, is to store the base URL as a string object so that each additional search string can be added to this as needed. Starting with the search term 'sneakers':

```python
base_url = 'https://www.amazon.com/s/ref=nb_sb_noss_1?url=search-alias%3Daps&field-keywords='

s_terms1 = 'sneakers'

search1 = base_url + s_terms1
```
