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

Use the **find_all** method to gather each **div class="s-item-container"** tag from the entire page. The result is a list of individual Beautiful Soup objects, each a section of the HTML markup that represents a single product. You can print Beautiful Soup objects using the **prettify** which produces a Unicode string in an easy to read format (truncated below).

```python
prod_li = page_soup.find_all('div', class_="s-item-container")

>>> print(prod_li[0].prettify())
<div class="s-item-container">
 <div class="a-row a-spacing-top-micro a-spacing-micro">
  <div class="a-row sx-badge-region">
   <div class="a-row a-spacing-large">
   </div>
  </div>
 </div>
 <div class="a-row a-spacing-base">
  <div aria-hidden="true" class="a-column a-span12 a-text-center s-position-relative">
  ...
</div>
```

Now that each of the product sections are stored as individual Beautiful Soup objects in a iterable list, we can use list comprehension to gather key data from each product section to begin building a dataframe. 

*Note: From this point forward, there will not be much detail about how to discover where certain elements are contained within the HTML. Just note that it is done through a combination of finding web elements using browser developer tool search functions and Beautiful Soup navigation techniques. Obviously, different pages are build differently by different developers, so this is the part of web scraping tasks that will vary from page to page and will require some amount of researching to get what you need.*

## Building a Dataframe

My preference for building a data set in Python is the Pandas dataframe. Specifically, I find it easiest to create a dictionary of lists (where each key is a dataframe variable with a list of values for every record) and then applying the **from_dict** method on that dictionary to create the dataframe. Each of the data gathering techniques, then, will involve gathering the data from the search return into separate lists.

To begin, make sure that all of the **div class="s-item-container"** tags in the list have some information contained within them as they are not of interest otherwise. The Beautiful Soup **contents** method will return each of the children of a Beutiful Soup object. The length of the **contents** call will be 0 if no children are present. Use this information along with list comprehension to clean up the main product list.

```python
prod_li = [bso for bso in prod_li if len(bso.contents) > 0]
```

### Product Title

Each product title is located inside an **h2** tag which is a child of each of the **div class="s-item-container"** tags pulled for the main product list. 

Calling a tag name on an object will return the first instance of that tag (Ex: **soup.a** returns the first **a** tag located inside of a Beautiful Soup object called soup). Subsequently, the **get_text** method returns only the text contained inside of of a given tag. Since the **h2** tag contains the product title text, you can use a combined approach in a list comprehension to build a new list conatining the product titles.

Also note (as you can see from the HTML snippet above that shows product title above), the **h2** tag will return the text "[Sponsored]". Since this information isn't necessarily a part of the desired data, you can remove as you pull in the title text using a pattern substitute function from the **re** package.

```python
import re

prod_title = [re.sub('\[Sponsored\]', '', bso.h2.get_text()) for bso in prod_li]

>>> prod_title[0]
"EpicStep Women's Canvas Shoes High Top Wedges High Heels Quilted Casual Fashion Sneakers"
```

  
