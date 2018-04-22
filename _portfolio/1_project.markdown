---
layout: post
title: Twitter Tool
description: How I used my web scraping tool in my own project
img: /img/Twitter.jpg
---
As part of my major ongoing project, "How Much Will You Earn Off Of Your Videogame?"
Twitter Tool has literally blown this project out of its socks. I was constantly hitting walls or coding block because I could not figure out how to get a large number of tweets. 

{% highlight python %}

class TweetCriteria:

    def __init__(self):
        self.maxTweets = 0

    def setUsername(self, username):
        self.username = username
        return self

    def setSince(self, since):
        self.since = since
        return self

    def setUntil(self, until):
        self.until = until
        return self

    def setQuerySearch(self, querySearch):
        self.querySearch = querySearch
        return self

    def setMaxTweets(self, maxTweets):
        self.maxTweets = maxTweets
        return self

    def setLang(self, Lang):
        self.lang = Lang
        return self

    def setTopTweets(self, topTweets):
        self.topTweets = topTweets
        return self


{% endhighlight %}

{% highlight python %}
class Tweet:

    def __init__(self):
        pass

{% endhighlight %}

{% highlight python %}

try:
    import http.cookiejar as cookielib
except ImportError:
    import cookielib
    
import urllib,json,re,datetime,sys
from pyquery import PyQuery

class TweetManager:
	
	def __init__(self):
		pass
		
	@staticmethod
	def getTweets(tweetCriteria, receiveBuffer=None, bufferLength=100, proxy=None):
		"""Description"""
		refreshCursor = ''
	
		results = []
		resultsAux = []
		cookieJar = cookielib.CookieJar()
		
		if hasattr(tweetCriteria, 'username') and (tweetCriteria.username.startswith("\'") or tweetCriteria.username.startswith("\"")) and (tweetCriteria.username.endswith("\'") or tweetCriteria.username.endswith("\"")):
			tweetCriteria.username = tweetCriteria.username[1:-1]

		active = True

		while active:
			json = TweetManager.getJsonReponse(tweetCriteria, refreshCursor, cookieJar, proxy)
			if len(json['items_html'].strip()) == 0:
				break

			refreshCursor = json['min_position']
			scrapedTweets = PyQuery(json['items_html'])
			#Remove incomplete tweets withheld by Twitter Guidelines
			scrapedTweets.remove('div.withheld-tweet')
			tweets = scrapedTweets('div.js-stream-tweet')
			
			if len(tweets) == 0:
				break
			
			for tweetHTML in tweets:
				tweetPQ = PyQuery(tweetHTML)
				tweet = Tweet()
				
				usernameTweet = tweetPQ("span:first.username.u-dir b").text()
				txt = re.sub(r"\s+", " ", tweetPQ("p.js-tweet-text").text().replace('# ', '#').replace('@ ', '@'))
				retweets = int(tweetPQ("span.ProfileTweet-action--retweet span.ProfileTweet-actionCount").attr("data-tweet-stat-count").replace(",", ""))
				favorites = int(tweetPQ("span.ProfileTweet-action--favorite span.ProfileTweet-actionCount").attr("data-tweet-stat-count").replace(",", ""))
				dateSec = int(tweetPQ("small.time span.js-short-timestamp").attr("data-time"))
				id = tweetPQ.attr("data-tweet-id")
				permalink = tweetPQ.attr("data-permalink-path")
				
				geo = ''
				geoSpan = tweetPQ('span.Tweet-geo')
				if len(geoSpan) > 0:
					geo = geoSpan.attr('title')
				
				tweet.id = id
				tweet.permalink = 'https://twitter.com' + permalink
				tweet.username = usernameTweet
				tweet.text = txt
				tweet.date = datetime.datetime.fromtimestamp(dateSec)
				tweet.retweets = retweets
				tweet.favorites = favorites
				tweet.mentions = " ".join(re.compile('(@\\w*)').findall(tweet.text))
				tweet.hashtags = " ".join(re.compile('(#\\w*)').findall(tweet.text))
				tweet.geo = geo
				
				results.append(tweet)
				resultsAux.append(tweet)
				
				if receiveBuffer and len(resultsAux) >= bufferLength:
					receiveBuffer(resultsAux)
					resultsAux = []
				
				if tweetCriteria.maxTweets > 0 and len(results) >= tweetCriteria.maxTweets:
					active = False
					break
					
		
		if receiveBuffer and len(resultsAux) > 0:
			receiveBuffer(resultsAux)
		
		return results
	
	@staticmethod
	def getJsonReponse(tweetCriteria, refreshCursor, cookieJar, proxy):
		"""Description"""
		url = "https://twitter.com/i/search/timeline?f=tweets&q=%s&src=typd&max_position=%s"
		
		urlGetData = ''
		
		if hasattr(tweetCriteria, 'username'):
			urlGetData += ' from:' + tweetCriteria.username
		
		if hasattr(tweetCriteria, 'querySearch'):
			urlGetData += ' ' + tweetCriteria.querySearch
		
		if hasattr(tweetCriteria, 'near'):
			urlGetData += "&near:" + tweetCriteria.near + " within:" + tweetCriteria.within
		
		if hasattr(tweetCriteria, 'since'):
			urlGetData += ' since:' + tweetCriteria.since
			
		if hasattr(tweetCriteria, 'until'):
			urlGetData += ' until:' + tweetCriteria.until
		
		if hasattr(tweetCriteria, 'topTweets'):
			if tweetCriteria.topTweets:
				url = "https://twitter.com/i/search/timeline?q=%s&src=typd&max_position=%s"
		url = url % (urllib.parse.quote(urlGetData), refreshCursor)
       
		headers = [('Host', "twitter.com"),
			('User-Agent', "Mozilla/5.0 (Windows NT 6.1; Win64; x64)"),
			('Accept', "application/json, text/javascript, */*; q=0.01"),
			('Accept-Language', "de,en-US;q=0.7,en;q=0.3"),
			('X-Requested-With', "XMLHttpRequest"),
			('Referer', url),
			('Connection', "keep-alive")]
		if proxy:
			opener = urllib.request.build_opener(urllib.request.ProxyHandler({'http': proxy, 'https': proxy}), urllib2.HTTPCookieProcessor(cookieJar))
		else:
			opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(cookieJar))
		opener.addheaders = headers

		try:
			response = opener.open(url)         
			jsonResponse = response.read()
		except:
			print ("Twitter weird response. Try to see on browser: https://twitter.com/search?q=%s&src=typd" % urllib.parse.quote(urlGetData))
			sys.exit()
			return
		dataJson = json.loads(jsonResponse)
		
		return dataJson		

{% endhighlight %}

{% highlight python %}
def finding_tweets(videogame):
    """Description"""
    name = []
    start_date = []
    tweet_date = []
    tweet_text = []
    assert isinstance(videogame[0],str)
    assert isinstance(videogame[1],str)
    assert isinstance(videogame[2],str)
    try:
        tweetCriteria = TweetCriteria().setQuerySearch(videogame[0]).setSince(videogame[1]).setUntil(videogame[2]).setMaxTweets(250)
        tweets = TweetManager.getTweets(tweetCriteria)
        for tweet in tweets:
            tweet_date.append(tweet.date)
            tweet_text.append(tweet.text)
    except:
        print(videogame[0],videogame[1],videogame[2], 'empty')
    df = pd.DataFrame(np.column_stack((tweet_date,tweet_text)))
    df['Name']= videogame[0] 
    df['start_date'] = videogame[1]
    df['end_date'] = videogame[2]
    return df
{% endhighlight %}
Since I had a long list of videogames with dates, I used multithreading to distribute the load. 
{% highlight python %}
from multiprocessing.pool import ThreadPool
import pickle
total_list = []
n=10
try:
    list_videogames = list_videogames.values
except:
    print('already array')
if not os.path.exists('Data/tweets.csv') or os.stat('Data/tweets.csv').st_size == 0:
    chunks = [list_videogames[i:i + n] for i in range(0, len(list_videogames), n)]
    for chunk in chunks:
        text_results = []
        pool=ThreadPool(6000)
        text_results = pool.map(finding_tweets,chunk)
        pool.close()
        pool.join()
        total_list = total_list + text_results
        with open('objs.pkl', 'wb') as f:
            pickle.dump([total_list], f)
    new_results = pd.DataFrame(np.vstack(total_list),columns=['date','text','name','start_date','end_date'])
    new_results.to_csv('/project/Springboard/Capstone_Project/Data/tweets.csv')
else:
    tweets_df = pd.read_csv('Data/tweets.csv')
    tweets_df = tweets_df.drop(tweets_df.columns[0], axis=1)
{% endhighlight %} 