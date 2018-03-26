## Introduction

The goal for this project is to take a list of search words, use each word or word combination in a basic search on [Amazon](https://www.amazon.com/), and pull key data (product name, price, rating) from the resulting page. The final result will be a structured data set that includes each of the key data points in addition to some basic metadata (order of results, search terms used) which can be used for analysis.

## GET HTML

The first step is to GET the web page HTML using the API of a PoolManager (via **urllib3**). As recommended by the **urllib3** [documentation](https://urllib3.readthedocs.io/en/latest/user-guide.html#ssl), SSL certificate verification should be used when making HTTPS requests. Mozilla's root certificate bundle is available via the certifi package.

```python
import urllib3
import certifi

http = urllib3.PoolManager(cert_reqs='CERT_REQUIRED', 
                           ca_certs=certifi.where())
```

To begin, open a web browser, navigate to the Amazon home page, and manually enter one of the specified searches in order to examine how the resulting URL points to the search results. For example, the resulting page from the search 'python books' (entered manually in the Amazon search box) has the following URL:

'https://www.amazon.com/s/ref=nb_sb_noss_1?url=search-alias%3Daps&field-keywords=python+books'

This indicates that the URL for a search result page is everything up to '...&field-keywords=' followed by the search word or words (separated by a '+' sign if the search contains more than one word). A couple of tests entering the above URL (adjusted for different search criteria) directly into the address bar of the browser will verify this assumption. 

The next step is to store the base of the above URL as a string object so that each search string can be added to this as needed. Starting with the search term 'sneakers', build a complete search URL as follows:

```python
base_url = 'https://www.amazon.com/s/ref=nb_sb_noss_1?url=search-alias%3Daps&field-keywords='

s_terms1 = 'sneakers'

search1 = base_url + s_terms1
```

With the first search URL compiled and the PoolManager created, use **urllib3**'s **urlopen** method on the PoolManager to GET the site data from the URL. 

The **urllib3** GET request creates an HTTPResponse object. Using the **BeautifulSoup** function on the **data** attribute of the HTTPResponse object creates a nested data representation of the HTML doc. This nested structure is key to navigating the HTML, locating the key data elements, and storing those elements in new objects for building the final data set.

*Note: The **BeautifulSoup** function requires an HTML parser. You can use the Python parser included in the standard library (**html.parser**) or an external parser. For this project, I'm using the **html5lib** parser.*

```python
from bs4 import BeautifulSoup
import html5lib

s_page = http.urlopen('GET', search1)

page_soup = BeautifulSoup(s_page.data, 'html5lib')
```

## Locate Key Data

The Beautiful Soup package provides a lot of useful functions that help with navigating to and from HTML tags indcluding finding parents, children, and siblings of tags as well as locating tags by CSS selectors, HTML tag types, or text within tags ([Details in the full documentation here](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)). The crucial piece in gathering data from a particular web site is understanding how to identify which HTML elements point to the data you need. *(Note: You can use developer tools in a browser to examine source code for a particular page to help get that information.)*

For this project, the key data for each product returned from the search is contained within **div** tags of the class **"s-item-container"**. The following is a snippet of that HTML markup for one of the products from the 'sneaker' search.

```html
<div class="s-item-container">
 <div class="a-row a-spacing-top-micro a-spacing-micro">
  ...
  <div class="a-row a-spacing-mini">
   ...
    <h2 class="a-size-base s-inline s-access-title a-text-normal" data-attribute="EpicStep Women's Canvas Shoes High Top Wedges High Heels Quilted Casual Fashion Sneakers" data-max-rows="0">
     <span class="a-offscreen">
      [Sponsored]
     </span>
     EpicStep Women's Canvas Shoes High Top Wedges High Heels Quilted Casual Fashion Sneakers
    </h2>
   ...
  </div>
  ...
 </div>
</div>
```

All of the HTML markup related to a single product is truncated in the outer '...' above (inside the first two **div** tags). This includes all of the key data needed to build the final data set. As an example, the snippet above shows part of the HTML markup where *product title* can be retrieved.

Use the **find_all** method to gather each **div class="s-item-container"** tag from the entire page. The result is a list of individual Beautiful Soup objects, each a section of the HTML markup that represents a single product.

```python
prod_li = page_soup.find_all('div', class_="s-item-container")
```
