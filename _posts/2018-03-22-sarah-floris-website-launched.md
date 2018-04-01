---
layout: default
title: Sarah Floris 
---


# Twitter Scraping Jupyter Notebook Interface

Twitter information is useful to analyze customers, human behavior, and other type of research. Unfortunately, Twitter restricts its access to the information via an API. A work around the API would be to webscrape the pages after a search. This jupyter notebook is an example of the Twitter Scraping Tool created on my Github page.  

Disclaimer: This tool still has its limits, so only the <a href="https://developer.twitter.com/en/docs/tutorials/twitters-enterprise-api-suite" target="_blank">Enterprise API</a> from Twitter has full access to Twitter data.

Having said that, let's begin!  

I will run the following code to verify that I meet all the requirement steps. The "!" means to run it on the terminal.
`
!pip install -r requirements.txt
`
Great! I meet all of the requirements. Now, I need to import the following modules so that I can run the code. 

`
import pandas as pd
import numpy as np
import re
`
The test data I will be using is part of my video games capstone project. The original data can be found on <a href="https://www.kaggle.com/rush4ratio/video-game-sales-with-ratings" target="_blank">Kaggle</a>.

`
#Importing the data and filling the empty cells with 0.
sales = pd.read_csv('/project/Springboard/Capstone Project/Data/Video_Games_Sales_as_at_22_Dec_2016 2.csv').fillna(value=0.0)

#Changing the limit to year of 2006 since that is the last year twitter has data
sales = sales[sales['Year_of_Release'] >= 2006.0]

#Deleting any weird symbols in the name, so that the name is clean for twitter search
sales['Name'] = [re.sub(r'[^\w\s]','',string) for string in sales['Name']] 
`

Now that the data is cleared, I can begin making my list for the search.

`
list_videogames = sales[['Name','Year_of_Release']].values
`
If your search is not that lengthy as mine, then you can use the following code.

`
length_df = pd.DataFrame({'length' : []})
length = []
for videogame in list_videogames[:1]: 
    search = '"'+videogame+'"' #need quoates because otherwise, query read as individual words or "include some of these words"
    path = '../tweet_files/'+videogame.replace(" ", "")+'.csv' #saving it to this file
    !python Exporter.py --since 2015-1-1 --querysearch $search --maxtweets 5000 --output $path
length_df = pd.read_csv(path,sep=';')
print(length_df)
`

However, the input I require has approximately 16000 videogames and dates. Thus, I created a function, so I can run it parallel. <br> <em> Note that the function I have created returns the text I searched, the number of total messages over that period, and the dates. </em>

`
def finding_tweets(listed):
    """I created this function so I could run Multithreading.
    Input: list of two values including the query and the date
    Output: list of the text and the count of the text"""
    import datetime
    from pathlib import Path
    search_text, search_year = listed
    length_df = pd.DataFrame({'length' : []})
    search = '"'+search_text+'"'
    filepath = '../tweet_files/'+search_text.replace(" ", "")+'.csv'
    search_date = datetime.date(int(search_year), 1, 1).isoformat()
    end_date = datetime.date(2016,12,22).isoformat()
    p = Path(filepath)
    if not p.exists() or p.stat().st_size == 0: #check if the file already exists or if its empty
        !python Exporter.py --querysearch $search --maxtweets 1000 --output $filepath
    try:
        length_df = pd.read_csv(filepath,sep=';')
        return (search_text, len(length_df), length_df['date'].values)
    except:
        length_df = ['0']
        return (search_text, len(length_df))
`

Now, it is time to set up the multithreading that speeds up the process. I first separated the list of videogames into chunks, so that I do not overflow the twitter page with search requests. Then, I take each chunk I have created and send it through 100 threads with ThreadPool. Lastly, I add the results to a list called "clean_list".

`
from multiprocessing.pool import ThreadPool
results = []
n = 10
clean_df = []
chunks = [list_videogames[i:i + n] for i in range(0, len(list_videogames), n)]
for chunk in chunks:
    pool=ThreadPool(100)
    results = pool.map(finding_tweets,chunk)
    pool.close()
    pool.join()
    clean_list = clean_list + results
`

Let's check the results. 

`
for row in clean_list:
    print(row)
`
Because I do not want to continually have to rerun the webscraper, I decided to save these values from clean_list to a  .csv file called "tweet.csv". 

`
pd.DataFrame(clean_df).to_csv('tweet.csv') 
`

Hope this helps with your adventure of exploring Twitter data! 












































