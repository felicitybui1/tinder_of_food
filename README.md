# Tinder Of Food
This project is a Flask interactive web application that displays a map of New York City and allows users to query it, along with a recommendation algorithm that matches suppliers to restuarants. The application uses a combination of Python, html, css, and Javascript. The data is stored using Apache Spark and MongoDB.

The website includes three route functions - /, /map, and /match (with GET and POST methods). 
The first route function, index(), renders a map template file and displays a map of New York City with some markers. It returns the index.html template with the map embedded. The map retrieves data from Apache Spark, which stores business data such as name, rating, price range, hours of operation. Then, for every restuarant the highlighted reviews are retrieved from the MongoDB database.

The map() function also renders a map template file with the same functionality as the index() function.
The post_map() function takes user input from the map form and queries a database to return the matching results. It then renders a new map with the matching markers.
The function uses the requests module to retrieve data from the database and create markers and popups on the map. 

The application also uses recommendation algorithms to match suppliers with restaurants using the post_match() function. Here, I created a recommendation algorithm that queries the Spark dabase based on the supplier profile information.

### **Import Libraries**

This script imports several Python libraries and checks if they are already installed. If a library is not installed, it is installed using pip. Then, several libraries are imported for use later in the script.

The importlib library is imported first to check if the other libraries are already installed. A list of libraries to check/install is defined as libraries. A loop is then used to iterate through each library in the libraries list. For each library, importlib.import_module() is used to try to import it. If the import is successful, a message is printed indicating that the library is already installed. If the import is unsuccessful due to an ImportError, the library is installed using pip and a message is printed indicating that the library has been installed.

After checking and installing the necessary libraries, several libraries are imported for use in the script. These include requests, json, pandas, findspark, pyspark, os, re, flask, folium, and ast.


```python
import importlib

# list of libraries to check/install
libraries = ['flask', 'folium', 'pyspark', 'PyDrive', 'findspark', 'werkzeug', 'pymongo']

# loop through libraries and check/install
for lib in libraries:
    try:
        importlib.import_module(lib)
        print(f"{lib} is already installed")
    except ImportError:
        #!pip install {lib}
        print(f"{lib} has been installed")

# import libraries
import requests
import json
import pandas as pd
import findspark
findspark.init()
from pyspark.sql import SparkSession
from pyspark import SparkContext
from pyspark.sql.functions import col
import pyspark.sql.functions as F
from pyspark.sql.functions import length
from pyspark.sql.functions import lit
from pyspark.sql.functions import array, array_contains
import os
import re
from flask import Flask, request, render_template, redirect, url_for, Markup
import folium
import ast
import datetime
import pymongo
from pymongo import MongoClient
```

    flask is already installed
    folium is already installed
    pyspark is already installed
    PyDrive has been installed
    findspark is already installed
    werkzeug is already installed
    pymongo is already installed


#### **Creating list of Zip Codes**
This code create a list of zip code representing all the zip codes of Manhatthan we will use to extract restaurant data.


```python
zip_codes = ['10001', '10002', '10003', '10004', '10005', '10006', '10007',
             '10009', '10010', '10011', '10012', '10013', '10014', '10016',
             '10017', '10018', '10019', '10021', '10022', '10023', '10024',
             '10025', '10026', '10027', '10028', '10029', '10030', '10031',
             '10032', '10033', '10034', '10035', '10036', '10037', '10038',
             '10039', '10040', '10044', '10069', '10103', '10119', '10128',
             '10162', '10165', '10170', '10173', '10199', '10279', '10280',
             '10282']
len(zip_codes)
```




    50



### **Get restaurants with Yelp Fusion API** (No need to run)
This script defines a function called get_restaurants that retrieves a list of restaurants from the Yelp API based on a given zip code. The function uses the requests library to make an HTTP GET request to the Yelp API with the appropriate headers and query parameters. The response is then parsed using the json library and converted to a Pandas DataFrame before being returned.

The get_restaurants function is called with the zip code 1, and the resulting DataFrame is assigned to a variable called restaurants. The script then loops through a list of zip_codes starting from the second item, and for each zip code, calls the get_restaurants function and concatenates the resulting DataFrame with the existing restaurants DataFrame using the pd.concat() function. The shape of the resulting DataFrame is printed for each iteration of the loop.


```python
# Set up API endpoint and headers
def get_restaurants(zip_code):
    url = 'https://api.yelp.com/v3/businesses/search'
    headers = {
        'Authorization': 'Bearer 0mkRcY_UEOS6NLS6zHwNkcm7yqpTKP2VufPM0EBwwCTKlt6W8u1dw5aBIbH4nWnJ6lU8PLRoJhQg1DgblFErBx_fLxVRNhF3j4-cODjk_HVDMuDdFiY6r0vQZ9chZHYx'
    }

    # Set up query parameters
    params = {
        'location': f"New York, NY {zip_code}",
        'categories': 'restaurants',
        'limit': 50
    }

    # Make API request
    response = requests.get(url, headers=headers, params=params)

    # Parse the response JSON
    data = json.loads(response.text)['businesses']
    data = pd.DataFrame(data)
    return data

restaurants = get_restaurants(1)
for zip_code in zip_codes[1:]:
    restaurant = get_restaurants(zip_code)
    restaurants = pd.concat([restaurants, restaurant])
    print(restaurants.shape)
```

#### **Drop Duplicates** (No need to run)
This script manipulates a Pandas DataFrame called restaurants. The first line drops duplicate rows in the DataFrame based on the alias column, using the drop_duplicates() method with the subset parameter set to 'alias' and the inplace parameter set to True. This modifies the restaurants DataFrame in place by removing any duplicate rows based on the alias column.

The second line resets the index of the DataFrame to be sequential integers starting from 0, using the reset_index() method with the drop parameter set to True to remove the old index column and the inplace parameter set to True to modify the restaurants DataFrame in place.

The final line returns the length of the modified restaurants DataFrame using the built-in len() function. This line of code simply calculates the number of rows in the DataFrame after duplicates have been removed and the index has been reset.


```python
restaurants.drop_duplicates(subset=['alias'],inplace=True)
restaurants.reset_index(drop=True, inplace=True)
len(restaurants)
```

#### **Extract Latitude and Longitude** (No need to run)
This script defines a function called extract_lat_long that takes a JSON string as input and extracts the latitude and longitude values from it. The function first replaces any single quotes in the JSON string with double quotes using the replace() method, then uses the json.loads() method to convert the string to a Python object. The function returns a tuple containing the latitude and longitude values extracted from the object using dictionary indexing.

The next line of code applies the extract_lat_long function to the coordinates column of the restaurants DataFrame using the apply() method. The resulting output is a new DataFrame with two columns called latitude and longitude, which are created by applying the pd.Series() method to the output of the extract_lat_long function.

The final line of code drops the original coordinates column from the restaurants DataFrame using the drop() method with axis=1 to indicate that the column should be dropped along the columns (i.e., the X-axis). The inplace parameter is set to True to modify the restaurants DataFrame in place.


```python
def extract_lat_long(json_str):
    json_obj = json.loads(str(json_str).replace("'", "\""))
    return json_obj['latitude'], json_obj['longitude']

# Apply the function to the 'location' column
restaurants[['latitude', 'longitude']] = restaurants['coordinates'].apply(extract_lat_long).apply(pd.Series)

# Drop the original 'location' column
restaurants.drop('coordinates', axis=1, inplace=True)
```

#### **Fill missing values** (No need to run)


```python
restaurants.fillna(0, inplace=True)
```

#### **Extract zip codes** (No need to run)
This script adds a new column called zip_code to the restaurants DataFrame. The values in this column are extracted from the location column of the DataFrame, which contains a JSON string describing the location of each restaurant.

The extraction is done using a regular expression (re.findall()) that matches the text "'zip_code': ' followed by one or more digits (\d+) followed by a closing single quote ('). This regular expression is applied to the location column using the apply() method and a lambda function.

The lambda function converts the x input (which is a row from the location column) to a string using the str() method, then applies the regular expression using re.findall() to extract the zip code. The [0] index is used to extract the first (and only) match, which is assumed to be the zip code for that row.


```python
restaurants['zip_code'] = restaurants['location'].apply(lambda x: re.findall(r"'zip_code': '(\d+)'", str(x))[0])
```

#### **Extract categories and location** (No need to run)
This script reads the 'categories' and 'location' columns of a pandas DataFrame called file1. It then processes these columns to create two new columns: 'foods' and 'zip'.

The 'categories' column contains strings that look like lists of dictionaries. To convert these strings to lists of dictionaries, the literal_eval() function from the ast module is used. Then, for each list of dictionaries, the script extracts the 'alias' values and adds them to a set of categories (cate) and a temporary list (temp). The temporary list is then appended to the categories list, resulting in a list of lists of categories for each row in the DataFrame.

The 'location' column also contains strings that look like dictionaries. The script uses literal_eval() to convert these strings to dictionaries and then extracts the 'zip_code' value. The 'zip_code' values are stored in a list called zips.

Finally, the script adds the categories list and the zips list as new columns to the file1 DataFrame, with column names 'foods' and 'zip', respectively


```python
l1 = file1['categories']
l2 = file1['location']

categories = []
zips = []

cate = set()

# print the resulting dictionary

for elems in l1:
    elems = literal_eval(elems)
    temp = []
    for elem in elems:
        temp.append(elem.get('alias'))
        cate.add(elem['alias'])
    categories.append(temp)

for elems in l2:
    elems = literal_eval(elems)
    zips.append(elems.get('zip_code'))
  

file1['foods'] = categories
file1['zip'] = zips
```

#### **Export dataframe to csv** (No need to run)


```python
restaurants.to_csv('restaurants.csv')
```

#### **Read Restaurants data**


```python
restaurants = pd.read_csv('restaurants.csv').drop(['Unnamed: 0'], axis=1)
restaurants.head()
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
      <th>alias</th>
      <th>name</th>
      <th>image_url</th>
      <th>is_closed</th>
      <th>url</th>
      <th>review_count</th>
      <th>categories</th>
      <th>rating</th>
      <th>coordinates</th>
      <th>transactions</th>
      <th>price</th>
      <th>location</th>
      <th>phone</th>
      <th>display_phone</th>
      <th>distance</th>
      <th>foods</th>
      <th>zip</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>veq1Bl1DW3UWMekZJUsG1Q</td>
      <td>gramercy-tavern-new-york</td>
      <td>Gramercy Tavern</td>
      <td>https://s3-media2.fl.yelpcdn.com/bphoto/f14WAm...</td>
      <td>False</td>
      <td>https://www.yelp.com/biz/gramercy-tavern-new-y...</td>
      <td>3303</td>
      <td>[{'alias': 'newamerican', 'title': 'American (...</td>
      <td>4.5</td>
      <td>{'latitude': 40.73844, 'longitude': -73.98825}</td>
      <td>['delivery']</td>
      <td>$$$$</td>
      <td>{'address1': '42 E 20th St', 'address2': '', '...</td>
      <td>1.212477e+10</td>
      <td>(212) 477-0777</td>
      <td>4698.218345</td>
      <td>['newamerican']</td>
      <td>10003</td>
    </tr>
    <tr>
      <th>1</th>
      <td>DGhWO1sUWydVeR5j5ZZaMw</td>
      <td>la-grande-boucherie-new-york-2</td>
      <td>La Grande Boucherie</td>
      <td>https://s3-media3.fl.yelpcdn.com/bphoto/b9URGc...</td>
      <td>False</td>
      <td>https://www.yelp.com/biz/la-grande-boucherie-n...</td>
      <td>2166</td>
      <td>[{'alias': 'french', 'title': 'French'}, {'ali...</td>
      <td>4.5</td>
      <td>{'latitude': 40.7626274, 'longitude': -73.9808...</td>
      <td>['delivery', 'pickup']</td>
      <td>$$$</td>
      <td>{'address1': '145 W 53rd St', 'address2': '', ...</td>
      <td>1.212511e+10</td>
      <td>(212) 510-7714</td>
      <td>7316.393738</td>
      <td>['french', 'steak', 'cocktailbars']</td>
      <td>10019</td>
    </tr>
    <tr>
      <th>2</th>
      <td>s3jou_L_LVYGkNHiuhjlew</td>
      <td>boucherie-west-village-new-york-3</td>
      <td>Boucherie West Village</td>
      <td>https://s3-media2.fl.yelpcdn.com/bphoto/-5TXMV...</td>
      <td>False</td>
      <td>https://www.yelp.com/biz/boucherie-west-villag...</td>
      <td>2213</td>
      <td>[{'alias': 'french', 'title': 'French'}, {'ali...</td>
      <td>4.5</td>
      <td>{'latitude': 40.733063, 'longitude': -74.0028772}</td>
      <td>['delivery', 'pickup']</td>
      <td>$$$</td>
      <td>{'address1': '99 7th Ave S', 'address2': '', '...</td>
      <td>1.212837e+10</td>
      <td>(212) 837-1616</td>
      <td>4505.017145</td>
      <td>['french', 'cocktailbars', 'steak']</td>
      <td>10014</td>
    </tr>
    <tr>
      <th>3</th>
      <td>nRO136GRieGtxz18uD61DA</td>
      <td>eleven-madison-park-new-york</td>
      <td>Eleven Madison Park</td>
      <td>https://s3-media1.fl.yelpcdn.com/bphoto/s_H7gm...</td>
      <td>False</td>
      <td>https://www.yelp.com/biz/eleven-madison-park-n...</td>
      <td>2390</td>
      <td>[{'alias': 'newamerican', 'title': 'American (...</td>
      <td>4.5</td>
      <td>{'latitude': 40.7416907417333, 'longitude': -7...</td>
      <td>[]</td>
      <td>$$$$</td>
      <td>{'address1': '11 Madison Ave', 'address2': '',...</td>
      <td>1.212889e+10</td>
      <td>(212) 889-0905</td>
      <td>5035.227660</td>
      <td>['newamerican', 'french', 'cocktailbars']</td>
      <td>10010</td>
    </tr>
    <tr>
      <th>4</th>
      <td>h37t9rA06Sr4EetJjKrfzw</td>
      <td>don-angie-new-york</td>
      <td>Don Angie</td>
      <td>https://s3-media2.fl.yelpcdn.com/bphoto/onJX6_...</td>
      <td>False</td>
      <td>https://www.yelp.com/biz/don-angie-new-york?ad...</td>
      <td>686</td>
      <td>[{'alias': 'italian', 'title': 'Italian'}, {'a...</td>
      <td>4.5</td>
      <td>{'latitude': 40.73778, 'longitude': -74.00197}</td>
      <td>['delivery']</td>
      <td>$$$</td>
      <td>{'address1': '103 Greenwich Ave', 'address2': ...</td>
      <td>1.212890e+10</td>
      <td>(212) 889-8884</td>
      <td>4954.078754</td>
      <td>['italian', 'newamerican']</td>
      <td>10014</td>
    </tr>
  </tbody>
</table>
</div>



#### **Initiate and configure Spark Session and Context**
This code sets up a SparkSession with the name "business" and a default parallelism of 20. The appName() method sets the name of the Spark application that will be shown in the Spark web UI. The config() method is used to configure Spark properties, such as the number of partitions or the memory usage. In this case, it sets the spark.default.parallelism configuration parameter to 20, which is the number of partitions that Spark will use by default for distributed operations.

The getOrCreate() method ensures that the session is either created or retrieved from the current SparkContext. The SparkContext.getOrCreate() method retrieves the existing SparkContext or creates a new one if none exists. The spark.version attribute is used to print the current version of Spark being used in the session.


```python
spark = SparkSession\
    .builder\
    .appName("business")\
    .config("spark.default.parallelism", 20)\
    .getOrCreate()
sc = SparkContext.getOrCreate(spark)
print("Using Apache Spark Version", spark.version)
```

    Setting default log level to "WARN".
    To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
    23/04/30 10:14:32 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable


    Using Apache Spark Version 3.4.0


#### **Create Restaurants Database**
This code uses Spark SQL to create a database with the name "restaurants". The CREATE DATABASE statement creates a new database if it doesn't already exist, and the IF NOT EXISTS clause ensures that the statement is only executed if the database does not already exist.

The db_name variable is used to specify the name of the database in the SQL statement. The spark.sql() method is used to execute the SQL statement within the SparkSession.


```python
db_name = "restaurants"
spark.sql(f"CREATE DATABASE IF NOT EXISTS {db_name}")
print(f"Database {db_name} created!")
```

    Database restaurants created!


#### **Read restaurants data into Spark**
This code reads a CSV file named "restaurants.csv" into a Spark DataFrame called restaurants_spark.

The spark.read.format("csv") statement specifies the format of the input file. The .options() method sets several options for reading the CSV file: header='true' specifies that the first row of the file contains the column names, inferschema='true' tells Spark to automatically infer the data types of each column, and treatEmptyValuesAsNulls='true' tells Spark to treat empty values as nulls.

The .load() method specifies the path of the CSV file to be read. In this case, the file is located at "/Users/cristianleo/Desktop/Data_Science/Datasets/restaurants.csv".

Once the file is read into the DataFrame, the columns attribute is used to print the names of all columns in the DataFrame.


```python
restaurants_spark = spark.read.format("csv") \
               .options(header='true', inferschema='true', treatEmptyValuesAsNulls='true') \
               .load("restaurants.csv")
restaurants_spark.columns
```




    ['Unnamed: 0',
     'id',
     'alias',
     'name',
     'image_url',
     'is_closed',
     'url',
     'review_count',
     'categories',
     'rating',
     'coordinates',
     'transactions',
     'price',
     'location',
     'phone',
     'display_phone',
     'distance',
     'foods',
     'zip']



### **Mongo DB**
#### **API Review Set Up**
This code defines a function get_reviews_highlights that takes a business alias as input and returns a list of reviews and their highlights from the Yelp API.

The function constructs the API endpoint URL with the input alias and sets the necessary headers for authentication with the Yelp API. It then sends a GET request to the endpoint and parses the JSON response to extract the list of reviews and their highlights. Finally, the function returns this list of reviews and highlights.


```python
def get_reviews_highlights(alias):
    url = "https://api.yelp.com/v3/businesses/{}/reviews?limit=20&sort_by=yelp_sort".format(alias)

    headers = {
        "accept": "application/json",
        'Authorization': 'Bearer wz7kVVrY8Klio2e20pRGe5QXHipLKFNUfnwr7cSOsaAxGD8Ab8ohn81oCrCK4oqjksRLn0QOtkHTPpkpnCvDv5Rutksc-d0QzZZiKC8J35YRnmYD57QG8GSe-Xw4ZHYx'
    }

    # Make API request
    response = requests.get(url, headers=headers)

    # Parse the response JSON
    data = response.json()
    return data['reviews']
```


This code defines a URI to connect to a MongoDB cluster hosted on the MongoDB Atlas cloud service, using the MongoClient class from the PyMongo library. The URI includes the MongoDB username and password, as well as the name of the cluster. Once the URI is defined, the client is started and connected to the MongoDB server.


```python
# uri (uniform resource identifier) defines the connection parameters 
uri = 'mongodb+srv://managingdata:123@cluster0.kvecqdp.mongodb.net/?retryWrites=true&w=majority'
# start client to connect to MongoDB server 
client = MongoClient(uri)

print("Connected!")
```

    Connected!


The first line of code creates a variable named db, which is a reference to a MongoDB database named restaurants_db, that was created by connecting to a MongoDB server using the MongoClient object previously defined with a specific uri.

The second line creates a variable named reviews_collection, which is a reference to a specific collection named Reviews within the restaurants_db database. A collection is a group of documents stored in MongoDB, which is equivalent to a table in a relational database.


```python
db = client['restaurants_db']
reviews_collection = db['Reviews']
```

#### **Extract aliases and reviews** (No need to run)
This code iterates over all the restaurant aliases in the restaurants dataframe, calls the get_reviews_highlights() function for each alias, and saves the resulting reviews in a list rev and the alias in a list al.

The try block tries to fetch the reviews for each alias using the get_reviews_highlights() function. If successful, the reviews are appended to the rev list and the alias to the al list.

If an error occurs while fetching the reviews, the except block catches the error and prints a message indicating that the reviews could not be fetched for the corresponding restaurant alias.


```python
rev = []
al = []

# Iterate over the all the restaurants aliases in the DB
for alias in restaurants['alias']:
    try:
        reviews = get_reviews_highlights(alias)
        rev.append(reviews)
        al.append(alias)

        print(f"Inserted reviews for restaurant {alias}")
    except Exception as e:
        print(f"Error fetching reviews for restaurant {alias}: {e}")
```

#### **Extract test from reviews and create a dictionary** (No need to run)

The code iterates over the list of reviews rev which contains the reviews for each restaurant fetched using the Yelp API.

For each restaurant reviews, it extracts the text of each review, concatenates them and stores the result in a list rev2.

Then, it loops through the list of restaurants aliases al and creates a new list rev3 containing a dictionary for each restaurant. The dictionary has two key-value pairs, the alias of the restaurant and the concatenated reviews for that restaurant.


```python
rev2 = []
rev3 = []

for elems in rev:
    s = ''
    for t in elems:
        s += t['text'] 
    rev2.append(s)
    
    


for i in range(len(al)):
    rev3.append({'alias' : al[i]  , 'reviews' : rev2[i]})
```

#### **Create JSON file** (No need to run)


```python
with open("reviews.json", "w") as outfile:
    for elems in rev3:   
        json.dump(elems, outfile)
```

#### **Iterate over dictionary and upload reviews in Mongo DB** (No need to run)


```python
for elems in rev3:
    reviews_collection.insert_one({
            "alias": elems['alias'],
            "reviews": elems['reviews'] })
```

#### **Check that the data are uploaded correctly**


```python
pipeline = [
       {"$match": { "alias":  "ippudo-ny-new-york-7"}},
    {"$project": {"reviews":1 , "_id":0}}
    
]
reviewString = list(reviews_collection.aggregate(pipeline))[0]['reviews']
reviewString
```




    'The vegetarian food selection here is superb. Great staff, great service. The servers help you with whatever you need.Great food and service, easily one the best ramen restraints in the city and recommend anyone visitingFresh food. At the bar you can see cooking area.The staff is friendly and helpful. They also have a great selection of beer and wine. The atmosphere is...'



#### **Set up Autocomplete API**

This function autocomplete(text) sends an API request to Yelp to obtain the ids of businesses that match a given text string in their name, category, or location.

The function takes a string as an argument named text.
The API endpoint and authentication headers are defined.
The query parameters are set with the text string, the locale parameter, and optional parameters like limit, location or categories.
A GET request is sent to the API endpoint using requests.get().
The response is converted from JSON format to a Python dictionary, and the list of aliases is extracted from the 'id' key of each business dictionary in the response.
Finally, the list of aliases is returned as the output of the function.


```python
# Set up API endpoint and headers
def autocomplete(text):
    #url = 'https://api.yelp.com/v3/businesses/search'
    url = 'https://api.yelp.com/v3/autocomplete'
    headers = {
        'Authorization': 'Bearer 0mkRcY_UEOS6NLS6zHwNkcm7yqpTKP2VufPM0EBwwCTKlt6W8u1dw5aBIbH4nWnJ6lU8PLRoJhQg1DgblFErBx_fLxVRNhF3j4-cODjk_HVDMuDdFiY6r0vQZ9chZHYx'
    }

    # Set up query parameters
    params = {
        #'location': f"New York, NY {zip_code}",
        'text':text,
        'locale': 'en_US',
        #'limit': 50
    }
    
    # Make API request
    businesses = requests.get(url, headers=headers, params=params).json()['businesses']
    aliases_autocomplete = [alias['id'] for alias in businesses]
    return aliases_autocomplete
```

#### **Render NYC map**

This code creates a map of New York City using the Folium library, and adds markers to the map representing restaurants in the city.

The folium.Map function is called to create the map object with a specified center location, zoom level, and boundaries. The bounds variable specifies the latitude and longitude coordinates that define the boundaries of the map.

The code then uses a for loop to iterate over the first 100 rows of a Pandas DataFrame restaurants. For each row, it extracts information about the restaurant such as name, image URL, categories, review count, rating, price, address, and phone number. It also queries a MongoDB collection reviews_collection to obtain the highlighted review for the restaurant.

The extracted information is used to construct an HTML string, which is passed as the popup parameter to the folium.Marker function. The HTML string is formatted with the information obtained for the restaurant, and includes CSS styling to format the popup content.

Finally, the folium.Icon function is called to create a marker icon, and the folium.Marker function is called to create a marker for the restaurant with the specified location, popup, and icon. The CSS styling for the popup is added to the map HTML using the folium.Element function.

The purpose of rendering the map before launching the website is to reduce the workload on the website and the waiting time to load the page.


```python
nyc_map = folium.Map(location=[40.7128, -74.0060], zoom_start=13, max_zoom=16, min_zoom=12.5)
bounds = [(40.699364, -74.025970), (40.787199, -73.910167)]
# Set the boundaries of the map
nyc_map.fit_bounds(bounds)

for row in range(100):
    categories_str = restaurants.iloc[row]['categories']
    categories_list = ast.literal_eval(categories_str)
    categories = [category['title'] for category in categories_list]
    categories = ", ".join(categories)

    try:
        pipeline = [
            {"$match": { "alias":  restaurants.iloc[row]['alias']}},
            {"$project": {"reviews":1 , "_id":0}}
            
        ]
        reviewString = list(reviews_collection.aggregate(pipeline))[0]['reviews']
        if len(reviewString) > 50:
            reviewString = reviewString[:100] + "..."
    except Exception:
        reviewString = "None"
    popup_html = """
    <div class="popup-container">
        <h3>{}</h3>
        <img src="{}" alt="Restaurant Image" style="width:200px; height=auto;">
        <p><b>Category:</b> {}</p>
        <p><b>Number of reviews:</b> {}</p>
        <p><b>Rating:</b> {}</p>
        <p><b>Price:</b> {}</p>
        <p><b>Address:</b> {}</p>
        <p><b>Phone:</b> {}</p>
        <p><b>Highlighted Review:</b> {}</p>
    </div></div>
    """.format(restaurants.iloc[row]['name'],
            restaurants.iloc[row]['image_url'],
            categories,
            restaurants.iloc[row]['review_count'],
            restaurants.iloc[row]['rating'],
            restaurants.iloc[row]['price'],
            json.loads(restaurants.iloc[0]['location'].replace("'", '"'))['address1'],
            restaurants.iloc[row]['display_phone'],
            reviewString)

    popup = folium.Popup(popup_html, max_width=250)
    icon = folium.Icon(icon='cutlery', prefix='fa', color='orange')

    folium.Marker(location=[json.loads(restaurants.iloc[row]['coordinates'].replace("'", '"'))['latitude'],
                            json.loads(restaurants.iloc[row]['coordinates'].replace("'", '"'))['longitude']],
                    popup=popup, icon=icon).add_to(nyc_map)

    css = """
    .popup-container {
        font-family: 'Open Sans', sans-serif;
        font-size: 14px;
        line-height: 1.5em;
        padding: 10px;
        background-color: white;
        box-shadow: 0 2px 6px rgba(0, 0, 0, 0.3);
        border-radius: 4px;
    }
    .popup-container h3 {
        margin: 0 0 5px;
        font-size: 20px;
        font-weight: 600;
        color: #5b5b5b;
    }
    .popup-container p {
        margin: 0 0 10px;
        font-size: 14px;
        color: #5b5b5b;
    }
    .popup-container img {
        display: block;
        margin: 0 auto;
        border-radius: 4px;
        box-shadow: 0 2px 6px rgba(0, 0, 0, 0.3);
    }
    .popup-container a {
        display: block;
        margin-top: 10px;
        font-size: 14px;
        color: #9b4dca;
        text-decoration: none;
        text-align: center;
        transition: color 0.2s ease-in-out;
    }
    .popup-container a:hover {
        color: #722fa8;
    }
"""

    # Add the CSS to the map HTML
    folium.Element(css).render()
```

# **Set up interactive Flask App**
This is a Flask app that provides an interface for users to search for restaurants in New York City. The code has several routes for different pages and functions.

The first route (`/`) is for the homepage, and it renders a template called `index.html` with an embedded map of NYC. The map is generated using Folium and converted to an HTML string before being passed to the template.

The second route (`/map`) is for the search results page, and it also renders a template called `map.html` with an embedded map of NYC. This map is also generated using Folium and converted to an HTML string before being passed to the template. However, this route also includes a form for users to search for restaurants. When the form is submitted, the data is processed and used to filter a Pandas dataframe of restaurant data. The filtered data is then used to generate a new map with markers representing the restaurants that match the search criteria.

The third route (`/slides`) renders a template called `slides.html`, which is presumably a slideshow or presentation of some kind.

The fourth route (`/map`) is the same as the second route, but with the addition of a POST method. This route is used to handle the form submission from the search results page. The form data is extracted from the request object and used to filter the restaurant data. The filtered data is then used to generate a new map with markers representing the restaurants that match the search criteria. If the search query returns more than 500 results, only the first 500 are displayed.

The code makes use of several libraries, including Flask, Folium, and Pandas. It also makes use of a `restaurants_spark` object, which is presumably a Spark dataframe containing restaurant data, as well as a `reviews_collection` object, which is presumably a MongoDB collection containing restaurant reviews. The `autocomplete` function is used to suggest restaurant names based on a user's search query.

The Flask web application then receives a request from the client at the URLs "/output", "/matching", and "/matching" with methods GET and POST. The main function of this web application is to filter a list of restaurants based on user input and present the matching results in a web page.

The function `output` returns a rendered HTML template called "output.html" when the client accesses the URL "/output". 

The function `matching` creates an empty file called "output.html" and returns a rendered HTML template called "matching.html" when the client accesses the URL "/matching".

The function `matching_post` is called when the client submits a POST request to the URL "/matching". This function extracts information from the form that the user fills out in the web page and then queries the list of restaurants to find matching results. The results are then displayed in a formatted HTML string that is included in a complete HTML file template called "matching_html". This file is then returned to the client as the response to the POST request.

Then the code defines a Flask route at the URL '/output' using the @app.route decorator. This route only accepts GET requests.

The re_output function is executed when the route is accessed. It first prints 'Updated' to the console, then returns a rendered HTML template called 'output.html'.

The last few lines use the if __name__ == '__main__': statement to ensure that the Flask app only runs if the Python script is run directly (as opposed to being imported as a module into another script). If the script is being run directly, it starts the Flask app on the local host at port 5001 using the app.run() method.


```python
app = Flask("Deliop - The Tinder of Food", static_folder='static')
app.config['TEMPLATES_AUTO_RELOAD'] = True
        
@app.route('/')
def index():
    
    # Convert the map to an HTML string
    nyc_map_html = nyc_map._repr_html_()

    # Render the template with the map embedded
    return render_template('index.html', nyc_map=nyc_map_html)

@app.route('/map')
def map():
    # Convert the map to an HTML string
    
    nyc_map_html = nyc_map._repr_html_()

    # Render the template with the map embedded
    return render_template('map.html', nyc_map=nyc_map_html)

@app.route('/slides')
def slides():
    # Convert the map to an HTML string

    # Render the template with the map embedded
    return render_template('slides.html')

@app.route('/map', methods=['GET','POST'])
def post_map():
    #arr = []
    city = request.form.get("city")
    cuisine = request.form.getlist("cuisine") #[0][1:-1].split(', ')
    cuisine = [word.strip('[], ') for sub_list in cuisine for word in sub_list.split(',')]
    reviews = request.form.getlist("reviews")
    rating = request.form.getlist("rating")
    price = request.form.getlist("Price")
    location = request.form.getlist("Location")
    search = request.form.get("search")

    #arr = [cuisine, reviews, rating, price, location]

    data = restaurants_spark.select('*')

    if len(search) == 0:
        if len(cuisine) != 0:
            data = data.filter(col("foods").rlike("|".join(cuisine))).distinct()
            
        if len(reviews) != 0:
            temp = reviews[0]
            if '>' in temp:
                data  = data.where((F.col('review_count') > 500)).distinct()
            else:
                nums = temp.split('-')
                n1 = float(nums[0])
                n2 = float(nums[1])
                data  = data.where((F.col('review_count') >= n1) & (F.col('review_count') <= n2)).distinct()
                
        if len(rating) != 0:
            rat = float(rating[0])
            data = data.where(F.col('rating') >= rat)

            
        if len(price) != 0:
            l = int(price[0])
            print(l)
            data = data.where(length(col('price')) == l)
            
        if len(location) != 0:
            z = location
            data  = data.filter(array_contains(array([lit(a) for a in z]), F.col('zip')))
        data_queried = data.toPandas()

    else:
        try:
            alias = autocomplete(search)
            data = data.toPandas()

            data_queried = data[data['id']==str(alias[0])]

            if len(alias) > 1:
                for restaurant_id in alias[1:]:
                    new_res = data[data['id']==str(restaurant_id)]
                    data_queried = pd.concat([data_queried, new_res])
        except Exception:
            pass
              
    if len(data_queried) > 500:
        data_queried = data_queried.head(500)
    # Render new map

    nyc_map_new = folium.Map(location=[40.7549, -73.9840], zoom_start=12, max_zoom=16, min_zoom=12.5)
    bounds = [(40.699364, -74.025970), (40.787199, -73.910167)]
    # Set the boundaries of the map
    nyc_map_new.fit_bounds(bounds)

    for row in range(len(data_queried)):
        try:
            categories_str = data_queried.iloc[row]['categories']
            categories_list = ast.literal_eval(categories_str)
            categories = [category['title'] for category in categories_list]
            categories = ", ".join(categories)

            try:
                pipeline = [
                    {"$match": { "alias":  data_queried.iloc[row]['alias']}},
                    {"$project": {"reviews":1 , "_id":0}}
                    
                ]
                reviewString = list(reviews_collection.aggregate(pipeline))[0]['reviews']
                if len(reviewString) > 50:
                    reviewString = reviewString[:100] + "..."
            except Exception:
                reviewString = "None"

            popup_html = """
            <div class="popup-container">
                <h3>{}</h3>
                <img src="{}" alt="Restaurant Image" style="width:200px; height=auto;">
                <p><b>Category:</b> {}</p>
                <p><b>Number of reviews:</b> {}</p>
                <p><b>Rating:</b> {}</p>
                <p><b>Price:</b> {}</p>
                <p><b>Address:</b> {}</p>
                <p><b>Phone:</b> {}</p>
                <p><b>Highlighted Review:</b> {}</p>
            </div></div>
            """.format(data_queried.iloc[row]['name'],
                    data_queried.iloc[row]['image_url'],
                    categories,
                    data_queried.iloc[row]['review_count'],
                    data_queried.iloc[row]['rating'],
                    data_queried.iloc[row]['price'],
                    json.loads(data_queried.iloc[row]['location'].replace("'", '"'))['address1'],
                    data_queried.iloc[row]['display_phone'],
                    reviewString)

            popup = folium.Popup(popup_html, max_width=250)
            icon = folium.Icon(icon='cutlery', prefix='fa', color='orange')

            folium.Marker(location=[json.loads(data_queried.iloc[row]['coordinates'].replace("'", '"'))['latitude'],
                                    json.loads(data_queried.iloc[row]['coordinates'].replace("'", '"'))['longitude']],
                            popup=popup, icon=icon).add_to(nyc_map_new)
        
            css = """
                .popup-container {
                    font-family: 'Open Sans', sans-serif;
                    font-size: 14px;
                    line-height: 1.5em;
                    padding: 10px;
                    background-color: white;
                    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.3);
                    border-radius: 4px;
                }
                .popup-container h3 {
                    margin: 0 0 5px;
                    font-size: 20px;
                    font-weight: 600;
                    color: #5b5b5b;
                }
                .popup-container p {
                    margin: 0 0 10px;
                    font-size: 14px;
                    color: #5b5b5b;
                }
                .popup-container img {
                    display: block;
                    margin: 0 auto;
                    border-radius: 4px;
                    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.3);
                }
                .popup-container a {
                    display: block;
                    margin-top: 10px;
                    font-size: 14px;
                    color: #9b4dca;
                    text-decoration: none;
                    text-align: center;
                    transition: color 0.2s ease-in-out;
                }
                .popup-container a:hover {
                    color: #722fa8;
                }
            """

            # Add the CSS to the map HTML
            folium.Element(css).render()
    
        except Exception:
            continue
    # Convert the map to an HTML string
    
    nyc_map_html_new = nyc_map_new._repr_html_()
    
    # Render new template with the new map
    return render_template('map.html', nyc_map=nyc_map_html_new)


@app.route('/output')
def output():
    return render_template('output.html')

@app.route('/matching')
def matching():
    with open('templates/output.html', 'w') as f:
            f.write("")
            f.flush()
    return render_template('matching.html')
    
@app.route('/matching', methods=['GET','POST'])
def matching_post():
    
    name = request.form.get('first_name')
    business = request.form.get('business')
    email = request.form.get('email')
    phone = request.form.get('phone')
    locations = request.form.getlist('locations')
    locations = [word.strip('[], ') for sub_list in locations for word in sub_list.split(',')]
    locations = [int(location) for location in locations]
    specialty = request.form.getlist('speciality')
    specialty = [word.strip('[], ') for sub_list in specialty for word in sub_list.split(',')]
    pricing_tier = request.form.get('pricing_tier')
    plan_selection = request.form.get('plan_selection')
    
    # Construct the response as a dictionary
    response = {
        'name': name,
        'business_name': business,
        'email': email,
        'phone': phone,
        'locations': locations,
        'specialty': specialty,
        'pricing_tier': pricing_tier,
        'plan_selection': plan_selection
    }
    print(response)

    data = restaurants_spark.select('*')

    if len(specialty) != 0:
        data = data.filter(col("foods").rlike("|".join(specialty))).distinct()

    if pricing_tier != '':
        l = len(pricing_tier)
        data = data.where(length(col('price')) == l)

    if len(locations) != 0:
        z = locations
        data  = data.filter(array_contains(array([lit(a) for a in z]), F.col('zip')))
        
    if plan_selection != '':
        a = plan_selection
        if a == '1':
            data = data.limit(10)
        if a == '2':
            data = data.limit(50)
        if a == '3':
            data = data.limit(100)

    data_queried = data.toPandas()

    content_output = """"""

    for row in range(len(data_queried)):
        try:
            categories_str = data_queried.iloc[row]['categories']
            categories_list = ast.literal_eval(categories_str)
            categories = [category['title'] for category in categories_list]
            categories = ", ".join(categories)

            content_output += f"""
                <div class="item item-type-zoom">
                    <a href="#" class="item-hover">
                        <div class="item-info">
                            <div class="headline">
                                {data_queried.iloc[row]['name']}
                                <div class="line"></div>
                                <div class="dit-line">Categories: {categories}</div>
                                <div class="dit-line">Categories: {data_queried.iloc[row]['review_count']}</div>
                                <div class="dit-line">Categories: {data_queried.iloc[row]['rating']}</div>
                                <div class="dit-line">Categories: {data_queried.iloc[row]['price']}</div>
                                <div class="dit-line">Categories: {json.loads(data_queried.iloc[row]['location'].replace("'", '"'))['address1']}</div>
                                <div class="dit-line">Categories: {data_queried.iloc[row]['display_phone']}</div>
                            </div>
                        </div>
                    </a>
                    <div class="item-img">
                        <img src="{data_queried.iloc[row]['image_url']}" alt="sp-menu" style="height: 275px; width: 325;">
                    </div>
                </div>
                """
        except Exception:
            continue

    matching_html = f"""<!DOCTYPE html>
<html lang="en">

<head>

    <!-- Basic -->
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">

    <!-- Mobile Metas -->
    <meta name="viewport" content="width=device-width, maximum-scale=1, initial-scale=1, user-scalable=0">

    <!-- Site Metas -->
    <title>Deliop</title>
    <meta name="keywords" content="">
    <meta name="description" content="">
    <meta name="author" content="">

    <!-- Site Icons -->
    <link rel="shortcut icon" href="http://localhost:5001/static/images/favicon.ico" type="image/x-icon" />
    <link rel="apple-touch-icon" href="http://localhost:5001/static/images/apple-touch-icon.png">

    <!-- Bootstrap CSS -->
    <link rel="stylesheet" href="http://localhost:5001/static/css/bootstrap.min3.css">
    <!-- Site CSS -->
    <link rel="stylesheet" href="http://localhost:5001/static/css/style.css">
    <!-- Responsive CSS -->
    <link rel="stylesheet" href="http://localhost:5001/static/css/responsive.css">
    <!-- color -->
    <link id="changeable-colors" rel="stylesheet" href="http://localhost:5001/static/css/colors/new_orange.css" />
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/leaflet.css" integrity="sha512-Ba/8Fbgp7R5jKqbEZy5RMfduJ+0qI8Q15WxgZ7btuZdCvdd7YLMz0d0XyDCQb1lDRKj8/h+LrLJxwodrOF6Ylw==" crossorigin="anonymous" referrerpolicy="no-referrer" />
    <!-- Modernizer -->
    <script src="http://localhost:5001/static/js/modernizer.js"></script> 


    <!--[if lt IE 9]>
    <script src="https://oss.maxcdn.com/libs/html5shiv/3.7.0/html5shiv.js"></script>
    <script src="https://oss.maxcdn.com/libs/respond.js/1.4.2/respond.min.js"></script>
    <![endif]-->
    
</head>

<body>
    
    <div id="site-header">
        <header id="header" class="header-block-top">
            <div class="container">
                <div class="row">
                    <div class="main-menu">
                        <!-- navbar -->
                        <nav class="navbar navbar-default" id="mainNav">
                            <div class="navbar-header">
                                <button type="button" class="navbar-toggle collapsed" data-toggle="collapse" data-target="#navbar" aria-expanded="false" aria-controls="navbar">
                                    <span class="sr-only">Toggle navigation</span>
                                    <span class="icon-bar"></span>
                                    <span class="icon-bar"></span>
                                    <span class="icon-bar"></span>
                                </button>
                                <div class="logo">
                                    <a class="navbar-brand js-scroll-trigger logo-header" href="http://localhost:5001/">
                                        <img src="http://localhost:5001/images/Deliopt_logo2.png" alt="">
                                    </a>
                                </div>
                            </div>
                            <div id="navbar" class="navbar-collapse collapse">
                                <ul class="nav navbar-nav navbar-right">
                                    <li class="active"><a href="http://localhost:5001/">Home</a></li>
                                    <li class="active"><a href="http://localhost:5001/matching">Match</a></li>
                                </ul>
                            </div>
                            <!-- end nav-collapse -->
                        </nav>
                        <!-- end navbar -->
                    </div>
                </div>
                <!-- end row -->
            </div>
            <!-- end container-fluid -->
        </header>
        <!-- end header -->
    </div>
    <!-- end site-header -->

    <div class="special-menu pad-top-100 parallax">
        <div class="container">
            <div class="row">
                <div class="col-lg-12 col-md-12 col-sm-12 col-xs-12">
                    <div class="wow fadeIn" data-wow-duration="1s" data-wow-delay="0.1s">
                        <h2 class="block-title color-white text-center"> Your Matches </h2>
                        <h5 class="title-caption text-center">  </h5>
                    </div>
                    <div class="special-box">
                        <div id="owl-demo">
    
                    {content_output}
        
                    </div>
                </div>
            </div>           
        </div>
    </div>
                

    <div id="footer" class="footer-main">
        <div class="footer-news pad-top-100 pad-bottom-70 parallax">
            <div class="container">
                <div class="row">
                    <div class="col-lg-12 col-md-12 col-sm-12 col-xs-12">
                        <div class="wow fadeIn" data-wow-duration="1s" data-wow-delay="0.1s">
                            <h2 class="ft-title color-white text-center"> Newsletter </h2>
                            <p> Subscribe our newsletter to receive latest news and exclusive offers</p>
                        </div>
                        <form>
                            <input type="email" placeholder="Enter your e-mail address">
                            <a href="#" class="new_orange-btn"><i class="fa fa-paper-plane-o" aria-hidden="true"></i></a>
                        </form>
                    </div>
                    <!-- end col -->
                </div>
                <!-- end row -->
            </div>
            <!-- end container -->
        </div>
        <!-- end footer-news -->
        <div class="footer-box pad-top-70">
            <div class="container">
                <div class="row">
                    <div class="footer-in-main">
                        <div class="footer-logo">
                            <div class="text-center">
                                <img src="http://localhost:5001/static/images/Deliopt_logo2.png" alt="" />
                            </div>
                        </div>
                        <div class="col-lg-3 col-md-3 col-sm-6 col-xs-12">
                            <div class="footer-box-a">
                                <h3>About Us</h3>
                                <p>Deliopt is your tinder of food. </p>
                                <p>It's match made in heaven. </p>
                                <ul class="socials-box footer-socials pull-left">
                                    <li>
                                        <a href="#">
                                            <div class="social-circle-border"><i class="fa  fa-facebook"></i></div>
                                        </a>
                                    </li>
                                    <li>
                                        <a href="#">
                                            <div class="social-circle-border"><i class="fa fa-twitter"></i></div>
                                        </a>
                                    </li>
                                    <li>
                                        <a href="#">
                                            <div class="social-circle-border"><i class="fa fa-google-plus"></i></div>
                                        </a>
                                    </li>
                                    <li>
                                        <a href="#">
                                            <div class="social-circle-border"><i class="fa fa-pinterest"></i></div>
                                        </a>
                                    </li>
                                    <li>
                                        <a href="#">
                                            <div class="social-circle-border"><i class="fa fa-linkedin"></i></div>
                                        </a>
                                    </li>
                                </ul>

                            </div>
                            <!-- end footer-box-a -->
                        </div>
                        <!-- end col -->
                        <div class="col-lg-3 col-md-3 col-sm-6 col-xs-12">
                            <div class="footer-box-c">
                                <h3>Contact Us</h3>
                                <p>
                                    <i class="fa fa-map-signs" aria-hidden="true"></i>
                                    <span>Columbia University in the City of New York</span>
                                </p>
                                <p>
                                    <i class="fa fa-mobile" aria-hidden="true"></i>
                                    <span>
                                    +1 (857) 260-0802
                                </span>
                                </p>
                                <p>
                                    <i class="fa fa-envelope" aria-hidden="true"></i>
                                    <span><a href="#">support@deliopt.com</a></span>
                                </p>
                            </div>
                            <!-- end footer-box-c -->
                        </div>
                        <!-- end col -->
                        <div class="col-lg-3 col-md-3 col-sm-6 col-xs-12">
                            <div class="footer-box-d">
                                <h3>Opening Hours</h3>

                                <ul>
                                    <li>
                                        <p>Monday - Friday </p>
                                        <span> 9:00 AM - 5:00 PM</span>
                                    </li>
                                </ul>
                            </div>
                            <!-- end footer-box-d -->
                        </div>
                        <!-- end col -->
                    </div>
                    <!-- end footer-in-main -->
                </div>
                <!-- end row -->
            </div>
            <!-- end container -->
            <div id="copyright" class="copyright-main">
                <div class="container">
                    <div class="row">
                        <div class="col-lg-12 col-md-12 col-sm-12 col-xs-12">
                            <h6 class="copy-title"> Copyright &copy; Deliopt is powered by <a href="#" target="_blank"></a> </h6>
                        </div>
                    </div>
                    <!-- end row -->
                </div>
                <!-- end container -->
            </div>
            <!-- end copyright-main -->
        </div>
        <!-- end footer-box -->
    </div>
    <!-- end footer-main -->

    <a href="#" class="scrollup" style="display: none;">Scroll</a>

    <section id="color-panel" class="close-color-panel">
        <!-- Colors -->
        <div class="segment">
            <h4 class="gray2 normal no-padding">Color Scheme</h4>
            <a title="orange" class="switcher new_orange-bg"></a>
            <a title="new_green" class="switcher new_green-bg"></a>
            <a title="new_red" class="switcher new_red-bg"></a>
            <a title="beige" class="switcher beige-bg"></a>
        </div>
    </section>

    <!-- ALL JS FILES -->
    
    <script src="http://localhost:5001/static/js/all.js"></script>
    <script src="http://localhost:5001/static/js/bootstrap.min.js"></script>
    <!-- ALL PLUGINS -->
    <script src="http://localhost:5001/static/js/custom.js"></script>

</body>

</html>
"""

    f = open('templates/output.html', 'w')
    f.write(Markup(matching_html))
    f.flush()
    f.close()
    print("overwritten")
    return redirect(url_for('output'))

    
@app.route('/output', methods=['GET'])
def re_output():  
    print('Updated')
    return render_template('output.html')

if __name__ == '__main__':
    app.run(host='localhost', port=5001)
```

     * Serving Flask app "Deliop - The Tinder of Food" (lazy loading)
     * Environment: production
    [31m   WARNING: This is a development server. Do not use it in a production deployment.[0m
    [2m   Use a production WSGI server instead.[0m
     * Debug mode: off


     * Running on http://localhost:5001/ (Press CTRL+C to quit)
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET / HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/css/bootstrap.min3.css HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/js/modernizer.js HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/css/style.css HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/css/responsive.css HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/css/colors/new_orange.css HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/css/font-awesome.min.css HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/css/bootstrap-datetimepicker.css HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/css/owl.carousel.css HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/css/bootstrap-select.min.css HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/css/owl.theme.css HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/css/slick.css HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/css/flaticon.css HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/css/normalize.css HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/css/animate.min.css HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/js/all.js HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/js/bootstrap.min.js HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/Prof.jpeg HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/js/custom.js HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/Deliopt_logo2.png HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/test3.png HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/Cris.jpeg HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/felicity.jpg HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/banner.jpg HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/Adam.jpg HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/fonts/nautilus-webfont.woff2 HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/fonts/fontawesome-webfont.woff2?v=4.7.0 HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/loader-animation.gif HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/Eric.jpeg HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/Samar.jpg HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/Yazan.jpeg HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/store.png HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/food.png HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/coffee.png HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/ingredients.jpg HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/peter_luger.jpg HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/team_bg.jpg HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/full-bg.png HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/blog_bg.jpg HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/footer_background.jpg HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/newsletter-bg.jpg HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:28] "GET /static/images/favicon.ico HTTP/1.1" 404 -
    127.0.0.1 - - [30/Apr/2023 10:16:31] "GET /static/images/icon_top.png HTTP/1.1" 304 -
    127.0.0.1 - - [30/Apr/2023 10:16:33] "GET /map HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:37] "POST /map HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:41] "GET /map HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:50] "GET /map HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:52] "GET /matching HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:16:54] "GET /static/fonts/glyphicons-halflings-regular.woff2 HTTP/1.1" 304 -


    {'name': 'Cristian Leo', 'business_name': 'Trabucco Sama S.r.l.', 'email': 'cristianleo120@gmail.com', 'phone': '3287141309', 'locations': [10011, 10003, 10010, 10016, 10031, 10032, 10033, 10034, 10040, 11101, 11102, 11103, 11104, 11105, 11106, 11354, 11355, 11356, 11357, 11358, 11360, 11361, 11362, 11363, 11364, 11365, 11366, 11367, 11412, 11423, 11432, 11433, 11434, 11435, 11436, 11691, 11692, 11693, 11694, 11697], 'specialty': ['cantonese', 'dimsum', 'foodstands', 'hotpot', 'shanghainese', 'streetvendors', 'szechuan', 'taiwanese', 'chinese', 'indpakm', 'indian', 'izakaya', 'japacurry', 'japanese', 'ramen', 'sushi'], 'pricing_tier': '$$', 'plan_selection': '3'}


    127.0.0.1 - - [30/Apr/2023 10:17:04] "POST /matching HTTP/1.1" 302 -
    127.0.0.1 - - [30/Apr/2023 10:17:04] "GET /output HTTP/1.1" 200 -
    127.0.0.1 - - [30/Apr/2023 10:17:04] "GET /images/Deliopt_logo2.png HTTP/1.1" 404 -
    127.0.0.1 - - [30/Apr/2023 10:17:04] "GET /static/images/special_menu_bg.jpg HTTP/1.1" 200 -


    overwritten
