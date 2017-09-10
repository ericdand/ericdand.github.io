---
layout: post
title: Script&#58; Mapping Probe Requests Using WiGLE
---

When searching for WiFi networks, devices send out *probe requests* for known
networks. By listening for these probe requests, we can make a list of the
SSIDs of networks that a device knows. [WiGLE](https://www.wigle.net) is an
open-source dataset of WiFi networks. By searching WiGLE for the SSIDs we hear
devices calling out to, we can create a map of where a person spends their
time.

I've created a script that listens for probe requests, queries WiGLE, and then
constructs Google Maps URLs from the coordinates returned by WiGLE. The script
is a proof of concept, meant to demonstrate what any organization with access
to a database like WiGLE (e.g. Apple, Google, Microsoft, spy agencies) can do
to ID and track people.

Most people do things like this on Linux, using the `aircrack-ng` suite of
tools. One of my goals in writing this script was to do it on a Mac, without
any extra hardware. So, my script uses the (built-in) `airport` utility to
control the WiFi antenna, and `tshark` to do the capturing.

# Listening for Probe Requests with tshark

Probe requests sometimes contain the SSID of a specific access point, but they
can also contain the "wildcard" SSID. The 802.11-2012 specification (section
8.2.4.3.4) states that "the value of all 1s is used to indicate the wildcard
BSSID." Probes using the wildcard SSID mean "all APs, please announce
yourselves to me." We're only interested in probes that contain an SSID, so we
want to filter these out. Wireshark prints this special wildcard value as
"Broadcast", but generally represents it as the empty string. We can therefore
filter based on whether the SSID is the empty string.

`tshark` is a command-line utility that comes with Wireshark. To capture probe
requests, we need to change our adapter to "monitor mode", which we can do with
`tshark -I`. We use a capture filter (`-f <filter>`) to only capture probe
requests, and a display filter (`-Y <filter>`) to only show non-wildcard
requests.

	tshark -I -a duration:30 -f 'subtype probe-req' -Y 'wlan.fcs_good && wlan_mgt.ssid != ""'

Note that `tshark` will complain if you try to use a display filter (`-Y`) and
write the output to a `.pcap` file (`-w <filename>`). The only filtering that
can be done while sniffing and writing is a capture filter (`-f <filter>`).
What a pain. I ended up parsing out the SSID from the standard output from
`tshark` using a regular expression.

# Querying WiGLE

A challenge comes in getting reconnected to the network after sniffing. MacOS
doesn't (seem to) have any command-line utility which can instruct the OS to
reconnect to a known network. The only way I found was to turn the WiFi adapter
off then back on again.

	networksetup -setairportpower en0 off
	networksetup -setairportpower en0 on

This will do whatever the adapter is configured to do when first turned on,
which is usually to connect to any known networks around. I ended up having to
build quite a bit of checking to get it all working. If the above fails, the
script will prompt the user to manually reconnect the Internet before
continuing, which I think is an acceptable fallback.

WiGLE has [an
API](https://api.wigle.net/swagger#!/Network_search_and_information_tools/search)
which can be used to query the database. You'll need to make an account and get
an API key, then the swagger page I've linked will help you build query URLs.
The API returns JSON, and luckily Python has a nice JSON parsing library
built-in that transforms JSON to Python dictionary objects.

Here's a snippet of Python that queries WiGLE and prints out coordinates:

{% highlight python %}
def query_wigle(params):
	url = 'https://api.wigle.net/api/v2/network/search?onlymine=false&freenet=false&paynet=false'
	for k in params.keys():
		url += '&' + k + '=' + params[k]
	try:
		f = popen("curl -H 'Accept:application/json' -u <YOUR API KEY HERE> '{}'".format(url))
		resp = json.loads(f.read())
		if not resp["success"]:
			print "ERROR from WiGLE: {}".format(resp["error"])
			return None
		print resp # DEBUG
		return resp
	except Exception as e:
		print "ERROR: {}".format(e)
		return None

data = query_wigle({ 'ssid': 'my cool wifi network', 'lastupdt': '20170101' })
print [(r['trilat'], r['trilong']) for r in data['results']]
{% endhighlight %}

# Too Many Points!

Pretty quickly, I realized that a lot of networks out there have the same
SSIDs. Typical WiGLE queries return dozens or even thousands of results. So, I
decided to do two things:

1. Only retrieve fresh data from WiGLE (within the last year or two).
2. Offer clustering analysis, and let the user choose to keep only some
   clusters.

The second of these two solutions took some work, and I'm still not thrilled by
the results. That said, it does help to cut down on the noise. Here's how it
works right now:

- If there are more than 10 results returned from WiGLE, ask the user if
  they want to use clustering.
- If they say yes, do k-means clustering, using fractions (1, 1/2, 1/3,
  1/4, and 1/5) of the square root of the number of results as k. (e.g. on
  100 results, we would try K=10, K=5, K=3, and K=2)
- For each clustering found, display the number of clusters (remember,
  k-means may return fewer than k clusters) and the average distance from
  the centroid, ask the user which one to use.
- Print map URLs for each cluster of the chosen clustering, then ask the
  user which cluster(s) they would like to keep for the final map.

Here's what it looks like in action:

	Querying WiGLE for SSID "my wifi network"...

	  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
									 Dload  Upload   Total   Spent    Left  Speed
	100 14033    0 14033    0     0   7228      0 --:--:--  0:00:01 --:--:--  9430

	26 results found for "my wifi network".
	Many results returned for SSID "my wifi network".
	Please choose what to do:
	(1) Do clustering analysis and choose which clusters to keep
	(2) Keep all results
	(3) Skip this SSID (keep none)
	
	>> 1
	
	Please choose a clustering.
	0: 5 clusters, fit score 12.7743174708
	1: 3 clusters, fit score 19.0982098411
	2: 2 clusters, fit score 36.5439548025
	3: 1 clusters, fit score 84.1351678636
	4: 1 clusters, fit score 84.1351678636
	
	>> 0
	
	Displaying clustering 0.

	Cluster 0:
	---------------------
	http://maps.google.com/maps/api/staticmap?size=640x640&maptype=roadmap&markers=color:red|41.906147,-87.63291168&markers=color:red|40.76970673,-73.9696579&markers=color:red|40.77045441,-73.96864319&markers=color:red|39.36798096,-74.44281769&markers=color:red|40.98980331,-82.7048111&markers=color:red|41.03591537,-73.92377472&markers=color:red|42.82785416,-84.21994781

	Cluster 1:
	---------------------
	http://maps.google.com/maps/api/staticmap?size=640x640&maptype=roadmap&markers=color:red|26.46385384,-80.14694977&markers=color:red|26.46376038,-80.14694977&markers=color:red|26.16716003,-80.31332397

	Cluster 2:
	---------------------
	http://maps.google.com/maps/api/staticmap?size=640x640&maptype=roadmap&markers=color:red|37.75439835,-122.16093445&markers=color:red|47.75764847,-122.21331787&markers=color:red|39.92105484,-105.04922485&markers=color:red|30.38583183,-97.67491913&markers=color:red|35.9800415,-115.15238953

	Cluster 3:
	---------------------
	http://maps.google.com/maps/api/staticmap?size=640x640&maptype=roadmap&markers=color:red|-6.23824596,106.78694916&markers=color:red|-6.2381978,106.78707123&markers=color:red|-35.33528137,149.08091736&markers=color:red|-16.90319824,145.7437439&markers=color:red|-16.92034531,145.77612305

	Cluster 4:
	---------------------
	http://maps.google.com/maps/api/staticmap?size=640x640&maptype=roadmap&markers=color:red|54.43619537,25.96686554&markers=color:red|56.1391716,8.96756458&markers=color:red|54.11406326,-3.22921538&markers=color:red|56.48365021,84.98144531&markers=color:red|56.45163727,9.40101719

	Which clusters would you like to keep? (format: '0, 1, 3')
	
	>> 0, 1, 2

	Map of all networks:
	http://maps.google.com/maps/api/staticmap?size=640x640&maptype=roadmap&markers=color:red|41.906147,-87.63291168&markers=color:red|40.76970673,-73.9696579&markers=color:red|40.77045441,-73.96864319&markers=color:red|39.36798096,-74.44281769&markers=color:red|40.98980331,-82.7048111&markers=color:red|41.03591537,-73.92377472&markers=color:red|42.82785416,-84.21994781&markers=color:red|26.46385384,-80.14694977&markers=color:red|26.46376038,-80.14694977&markers=color:red|26.16716003,-80.31332397&markers=color:red|37.75439835,-122.16093445&markers=color:red|47.75764847,-122.21331787&markers=color:red|39.92105484,-105.04922485&markers=color:red|30.38583183,-97.67491913&markers=color:red|35.9800415,-115.15238953

# K-Means Clustering

A few notes on clustering, because I had to learn how to do it for this project.

After some Googling, I found
[this](https://datasciencelab.wordpress.com/2013/12/12/clustering-with-k-means-in-python/)
excellent resource on k-means clustering. I aped the code from there into the
script, and it worked mostly fine. The one thing the article doesn't talk about
is what a good value for k is. 

After some experimentation and an actually decent [Wikipedia
article](https://en.wikipedia.org/wiki/Determining_the_number_of_clusters_in_a_data_set)
on the subject, I settled on k being the square root of the number of points,
and then trying some integer fractions of that number, as described earlier.

I also added some logic to the clustering algorithm to merge together any
clusters whose centres were within 10 degrees (whose distance changes depending
on where on the Earth it's measured, since degrees of longitude change as you
go north and south, but it's a bit more than 11,000 km at the equator) of each
other. I may still want to play with this setting.

# The Script

You can find the script on [my
GitHub](https://www.github.com/ericdand/security/tree/master/wifi-probe-reqs-poc.py). It
comes as-is, and isn't guaranteed to work at all. 
