## Introduction

The goal for this project is to take a specified list of search word(s), use each in a basic search on [Amazon](https://www.amazon.com/), and pull key data (product name, price, rating) from the resulting page. The final result will be structured data that includes each of the key data points in addition to some basic metadata (order of results, search terms used) which can be used for analysis.

## GET HTML

The first step is to GET the web page HTML using the API of a PoolManager (using **urllib3**). As recommended by the **urllib3** [documentation](https://urllib3.readthedocs.io/en/latest/user-guide.html#ssl), SSL certificate verification should be used when making HTTPS requests. Mozilla's root certificate bundle is available via the certifi package.

```python
import certifi
import urllib3

http = urllib3.PoolManager(cert_reqs='CERT_REQUIRED', 
                           ca_certs=certifi.where())
```

Starting at the home page for Amazon, manually enter one of the specified searches in order to examine the URL from the resulting page. For example, the search 'python books' (entered manually in the Amazon search tool) results in the following URL:

'https://www.amazon.com/s/ref=nb_sb_noss_1?url=search-alias%3Daps&field-keywords=python+books'

This indicates that the base URL for the search results page is everything up to '...&field-keywords=' followed by the search terms (separated by a '+' sign if search contains more than one word). A couple of tests typing different search terms directly into the URL in this fashion will verify this assumption. 

The next step is to store the base URL as a string object so that each additional search string can be added to this as needed. Starting with the search term 'sneakers', build a complete search URL as follows:

```python
base_url = 'https://www.amazon.com/s/ref=nb_sb_noss_1?url=search-alias%3Daps&field-keywords='

s_terms1 = 'sneakers'

search1 = base_url + s_terms1
```

With the first search URL compiled and the PoolManager created, use **urllib3**'s **urlopen** method on the PoolManager to GET the site data from the URL. 

The **urllib3** GET request creates an HTTPResponse object. Accessing the **data** property of this object and runnning this result through the Beautiful Soup package creates a nested data representation of the HTML doc. This nested structure is key to navigating the HTML, locating the key data elements, and storing those elements in new objects for building the final data set.

*Note: The **BeautifulSoup** function requires an HTML parser. You can use the Python parser included in the standard library (**html.parser**) or an external parser. For this project, I'm using the **html5lib** parser.*

```python
from bs4 import BeautifulSoup
import html5lib

s_page = http.urlopen('GET', search1)

page_soup = BeautifulSoup(s_page.data, 'html5lib')
```
