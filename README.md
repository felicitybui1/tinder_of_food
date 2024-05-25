# Tinder of Food

This project is a Flask interactive web application that displays a map of New York City and allows users to query it, along with a recommendation algorithm that matches suppliers to restuarants. The application uses a combination of Python, html, css, and Javascript. The data is stored using Apache Spark and MongoDB.

The website includes three route functions - /, /map, and /match (with GET and POST methods). The first route function, index(), renders a map template file and displays a map of New York City with some markers. It returns the index.html template with the map embedded. The map retrieves data from Apache Spark, which stores business data such as name, rating, price range, hours of operation. Then, for every restuarant the highlighted reviews are retrieved from the MongoDB database.

The map() function also renders a map template file with the same functionality as the index() function. The post_map() function takes user input from the map form and queries a database to return the matching results. It then renders a new map with the matching markers. The function uses the requests module to retrieve data from the database and create markers and popups on the map.

The application also uses recommendation algorithms to match suppliers with restaurants using the post_match() function. Here, I created a recommendation algorithm that queries the Spark dabase based on the supplier profile information.

## Import Libraries
This script imports several Python libraries and checks if they are already installed. If a library is not installed, it is installed using pip. Then, several libraries are imported for use later in the script.

The importlib library is imported first to check if the other libraries are already installed. A list of libraries to check/install is defined as libraries. A loop is then used to iterate through each library in the libraries list. For each library, importlib.import_module() is used to try to import it. If the import is successful, a message is printed indicating that the library is already installed. If the import is unsuccessful due to an ImportError, the library is installed using pip and a message is printed indicating that the library has been installed.

After checking and installing the necessary libraries, several libraries are imported for use in the script. These include requests, json, pandas, findspark, pyspark, os, re, flask, folium, and ast.

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
