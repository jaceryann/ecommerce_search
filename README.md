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

## Extract Data

Ultimately, this project will end up with a single data set: a Pandas dataframe. This will be created by applying the **from_dict** method on a dictionary of lists (where each dictionary key is a dataframe variable with a list containing the values for every record). Each of the data gathering techniques, then, will involve gathering the data from the search return into separate lists.

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

### Product Price

The price extraction is less straight forward than the title, but it provides a good example of how more complicated web design can make getting information more difficult and some approaches you can take to get there.

In the case of Amazon, the pricing information is contained within a **span** tag, but the classes vary. In addition, sometimes there are price ranges listed, and sometimes there is no price listed at all. And for even further complication, prices are rarely created as a simple string. Take, for example, the Beautiful Soup object, **ex_soup** which was extracted and stored from the working product (EpicStep Women's Canvas Shoes ...), which is a typical price setup.

```python
>>> print(ex_soup.prettify())
<span class="sx-price sx-price-large">
  <sup class="sx-price-currency">
    $
  </sup>
  <span class="sx-price-whole">
    50
  </span>
  <sup class="sx-price-fractional">
    99
  </sup>
</span>
```

If you were to **get_text** from this soup element, you would end up with line breaks and no decimal point. It would be easy enough to clean up this text to make it look more like a price tag, but these rules would not necissarily work for all of the product pricing. Take a look at a **ex2_soup** and **ex3_soup** below which were pulled from the same product list but for two different products:

```python
>>> print(ex2_soup.prettify())
<span class="sx-price sx-price-large">
 <sup class="sx-price-currency">
  $
 </sup>
 <span class="sx-price-whole">
  24
 </span>
 <sup class="sx-price-fractional">
  99
 </sup>
 <span class="sx-dash-formatting">
  -
 </span>
 <sup class="sx-price-currency">
  $
 </sup>
 <span class="sx-price-whole">
  36
 </span>
 <sup class="sx-price-fractional">
  99
 </sup>
</span>

>>> print(ex3_soup.prettify())
<span class="a-size-base a-color-base s-price">
 $9.99
</span>
```

In the first case above, the price is contained inside similar tags in the same fashion as the previous example, but for a price range instead of a single price (this means two price tags separated by a '-'). In the second, the **span** tag class is different and the price is a simple string that is not divided up into separate 'whole' and 'fractional' class tags. All of these factors will need to be accounted for as prices are being extracted. 

A useful aspect of Beautiful Soup is that you can create a custom function that returns **True** or **False** based on custom criteria and pass that function to **find_all**. You can also use a regular expression object for **find_all** arguments (such as **class_**, **string**, **href**, etc.). The code below applies both of these techniques to find prices based on the following criteria:

1. Look for **span** tag that has one of the following:
   * A child tag named **sup**
   * A string containing the '$' character
   * At least one child **span** tag of the class 'a-letter-space'
2. Parent **span** tag class should be one of the following:
   * 'sx-price'
   * 'sx-price-large'
   * 'a-size-base'
   * 'a-color-base'

```python
def has_price_char(tag):
    test2 = tag.findChild('sup') is not None
    test3 = tag.string is not None and '$' in tag.string
    test4 = len(tag.select('span.a-letter-space')) > 0
    return tag.name == 'span' and (test2 or test3 or test4)


regx_pr_cl = re.compile('(sx-price|sx-price-large|a-size-base|a-color-base)')

pr_tag = [bso.find(has_price_char, class_=regx_pr_cl) for bso in prod_li]
```

The **pr_tag** list should contain a single soup object for each of the instances where price was available and nothing where price was not available. *Note: Keeping the 'nothing' instances in the list maintains the record order for the final data set.*

The next step is to **get_text** from each of the items in **pr_tag** and clean up the text where needed. The code below shows before and after sample text for each of the different steps (for the three different price formats exemplified above: assume pr_tag[0] == ex_soup, pr_tag[1] == ex1_soup, and pr_tag[2] == ex2_soup).

```python
# first, get_text from pr_list: use split and join methods to remove white space
# store '' in list if no soup object available to maintain record length and order
prod_price = ["".join(bso.get_text().split()) if bso else '' for bso in pr_tag] 

>>> prod_price[0]
'$5099'

>>> prod_price[1]
'$2499-$3699'

>>> prod_price[2]
'$9.99'

# next, add in decimals where missing for single price items and top of range prices
prod_price = [prstr[:-2] + '.' + prstr[-2:] if '.' not in prstr else prstr for prstr in prod_price]

>>> prod_price[0]
'$50.99'

>>> prod_price[1]
'$2499-$36.99'

>>> prod_price[2]
'$9.99'

# finally, for price ranges, add decimal to bottom of range price
# create function for process to improve readability
def add_dot_rng(prstr):
    prstr = prstr[:(prstr.find('-') - 2)] + '.' + prstr[(prstr.find('-') - 2):]
    return prstr


prod_price = [add_dot_rng(prstr) if '-' in prstr and prstr.count('.') < 2 else prstr for prstr in prod_price]

>>> prod_price[0]
'$50.99'

>>> prod_price[1]
'$24.99-$36.99'

>>> prod_price[2]
'$9.99'
```

### Product Rating

The rating for each product, if available, is contained within a **span** tag of class 'a-icon-alt'. However, this class of **span** is also used for other icons so in order to differentiate, look for the word 'star' in the text. *Note: Again, maintain blanks where rating not available to keep lists the same length and order.*

```python
prod_rate = [bso.find('span', string=re.compile('star'), class_='a-icon-alt') for bso in prod_li]

prod_rate = [bso.get_text() if bso else '' for bso in prod_rate]

>>> prod_rate[0]
'3.3 out of 5 stars'
```

## Dataframe Build

Now that all of the key data is collected into separate lists, you can add these lists into a single dictionary of lists which will be converted into a Pandas dataframe.

In addition, this is a good time to add some additional data points that might be useful for analysis:
  * list_order: sequential integers that will show the order in which products appeared at the time of search
  * search_parameters: the search terms used to find the item (useful for building a bigger dataframe from multiple searches)

```python
final_data = {'list_order': list(range(1, len(pr_list) + 1)), 
              'product_title': prod_title, 
              'price': prod_price, 
              'rating': prod_rate, 
              'search_parameters': [s_terms_1] * len(pr_list)}

import pandas as pd

final_df = pd.DataFrame.from_dict(final_data)
```
