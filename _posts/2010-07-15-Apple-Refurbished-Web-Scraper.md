---
layout: post
title: Apple Refurbished Web Scraper
---

I've been looking for a refurbished mac for development recently and found myself leaving the refurbished site window open in the background and constantly checking the page. What a waste of time. Solution: write a web scraper to do the work for me. Below is a super simple scraper I wrote up that looks at Apple's refurbished page and scrapes for search terms (e.g. Mac mini, iMac, etc.). If new computers are listed that match your criteria it will shoot off an email (using a gmail account) to you with the listings description, price, and product URL.

{% highlight python %}
"""
Author: Daniel McGraw (@danielmcgraw, danielmcgraw.com, danielmcgraw.tumblr.com)
Description: A scraper used to find refurbished Apple computers.
"""

import re
import time
import urllib
import smtplib

from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText

from BeautifulSoup import BeautifulSoup

class RefurbScraper(object):

    def __init__(self, fromAddr, fromPass, toAddr, productName, interval):
        self.fromAddr = fromAddr
        self.fromPass = fromPass
        self.toAddr = toAddr
        self.productName = productName
        self.interval = interval
        self.activeURLList = []
										 		    
    def getSource(self, url):
    	page = urllib.urlopen(url)
        source = page.read()
        return source
														    
    def parseSource(self, source):
    	soup = BeautifulSoup(source)
        products = soup.findAll(re.compile('^table'))
        productList = []
        for product in products:
            secondA = product.findAll(re.compile('^a'))[1]
            productURL = 'http://store.apple.com' + secondA.attrs[0][1]
            productName = secondA.contents[0].lstrip().rstrip()
            span = product.findAll(re.compile('^span'))[0]
            productPrice = span.contents[0]
            productList.append((productName, productPrice, productURL))
        return productList

    def filterList(self, productList):
        newProducts = []
        print self.activeURLList
        temp = self.activeURLList
        self.activeURLList = []
        str = re.compile('^Refurbished %s' % self.productName)
        products = filter(lambda product: re.match(str, product[0]), productList)
        for product in products:
            if product[2] not in temp:
                newProducts.append(product)
            self.activeURLList.append(product[2])
        print self.activeURLList
        print
        return newProducts
	
    def sendEmail(self, products):
        if products:
            msg = MIMEMultipart('alternative')
            msg['Subject'] = "Refurbished %s's" % self.productName
            msg['From'] = self.fromAddr
            msg['To'] = self.toAddr

            body = "Refurbished %s's:\n\n" % self.productName
            for product in products:
                body += "\t%s\n\t%s\n\t%s\n\n" % (product[0], product[1], product[2])
            body += "This has been an automated email from Daniel McGraw's Apple Referb Scraper.\n"
            body += "Follow him on twitter(@danielmcgraw), tumblr(danielmcgraw.tumblr.com), or his blog(danielmcgraw.com)."
            msg.attach(MIMEText(body, 'plain'))
	
            print msg
		
            # Use gmail to send email.
            smtp = smtplib.SMTP('smtp.gmail.com', 587)
            smtp.ehlo()
            smtp.starttls()
            smtp.ehlo()
            smtp.login(self.fromAddr[:-9], self.fromPass)
            smtp.sendmail('<Refurbished Mac Scraper>%s' % self.fromAddr, self.toAddr, msg.as_string())
            smtp.close()
	
    def loop(self):
        while True:
            source = self.getSource('http://store.apple.com/us/browse/home/specialdeals/mac')
            productList = self.parseSource(source)
            products = self.filterList(productList)
            self.sendEmail(products)
            time.sleep(float(self.interval))
{% endhighlight %}

Usage is simple. Copy the script above into a file named AppleRefurbScraper, or anything else you want for that matter, but that's the name I'll be using. Then from a prompt start up python.
{% highlight python %}
>>> import AppleRefurbScraper
>>> scraper = AppleRefurbScraper.RefurbScraper('Gmail from address', 'Gmail from password', 'To address', 'search term', 'loop delay time in seconds')
>>> scraper.loop()
{% endhighlight %}

The search terms are case sensitive. The applicable search terms are:
    MacBook
    MacBook Air
    MacBook Pro
    Mac mini
    iMac
    Mac Pro
    Xserve
Also note that I use regex to match the search term so feel free to use a regex string as the search term.

If you like this project please subscribe to my <a href="http://danielmcgraw.com/feed/">feed</a>, and follow me on <a href="http://twitter.com/danielmcgraw">twitter</a> or <a href="http://danielmcgraw.tumblr.com/">tumblr</a> and say hi.