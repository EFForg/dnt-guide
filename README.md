# EFF HOW TO IMPLEMENT DNT GUIDE

This repository contains the guide for implementing EFF's [DNT Policy](https://www.eff.org/dnt-policy). We want your contributions and invite you to use it as a  space to share advice on web privacy engineering. If you have suggestions for other DNT-compliant service providers, please submit them. We are also looking for configurations for Windows servers to limit log collection (we are providing example code for Nginx, Apache and Logrotate). In the future, EFF will add sections dedicated to advertising and commenting systems.

## Warning
The configurations in this guide are examples and recommendations but you must exercise care when applying any changes in your configuration and test them beforehand.

## How to contribute to this guide

You can contribute to this guide via GitHub issues and pull requests.

## Table of Contents

- [1. Introduction: Why do you want to keep less data and respect Do Not Track?](#1-introduction-why-do-you-want-to-keep-less-data-and-respect-do-not-track)
- [2. Unique Identifiers and DNT Users](#2-unique-identifiers-and-dnt-users)
  - [Example: Database schema for a service that honors DNT:1 for logged in users](#example-database-schema-for-a-service-that-honors-dnt1-for-logged-in-users)
- [3. How to Limit Web Server Logs](#3-how-to-limit-web-server-logs)
  - [Apache](#apache)
  - [Nginx](#nginx)
  - [Logrotate](#logrotate)
- [4. Ensure Compliance by Third Parties](#4-ensure-compliance-by-third-parties)
- [5. How to Embed Content from Third Parties](#5-how-to-embed-content-from-third-parties)
  - [a. Embed.ly](#a-embedly)
  - [b. Shariff](#b-shariff)
- [6. Designing anonymized data structures](#6-designing-anonymized-data-structures)
  - [A. Common Anonymized Data Structures](#a-common-anonymized-data-structures)
  - [B. A Toolbox of Technical Resources for Designing Anonymized Data Structures](#b-a-toolbox-of-technical-resources-for-designing-anonymized-data-structures)
- [7. Analytics](#7-analytics)
- [8 Handling OCSP In a Privacy Preserving Manner](#8-handling-ocsp-in-a-privacy-preserving-manner)
- [9. DNT-Compliant Cloud Service Providers](#9-dnt-compliant-cloud-service-providers)
- [10. How to get Consent Permission for Additional Data Retention](#10-how-to-get-consent-permission-for-additional-data-retention)
- [11. Other Exceptions to the policy](#11-other-exceptions-to-the-policy)
- [12. How to Assert DNT Compliance](#12-how-to-assert-dnt-compliance)

## 1. Introduction: Why do you want to keep less data and respect Do Not Track?
This document describes ways to configure your website and web applications to retain less data, and to respect EFF’s [Do Not Track policy](https://www.eff.org/dnt-policy). This is an opt-out mechanism driven by the DNT: 1 headers that most modern browsers send if users enable a “Do Not Track” opt-out.[1]

The Do Not Track signal enables users to reliably and persistently transmit their user preference and provides that data to sites in a standardized form. 

Why would you want to honour a DNT request? There are several reasons:

1. Because you or your employer are committed to privacy as a goal, whether as a matter of principle or in order to mitigate the risk that the data you retain will be hacked, leaked, seized by a government, or subpoenaed in the course of civil litigation

2. Because you’re providing some kind of third-party widget or service, and you don’t want it blocked by tracker-blocking software. Several popular tracker- and ad-blocking tools ([Disconnect](https://disconnect.me/),[ Privacy Badger](https://www.eff.org/privacybadger), [AdNauseam](https://adnauseam.io/) now check for compliance with EFF DNT, and block third-party ads and analytics tools if they are not DNT compliant.

3. Because your website uses a lot of other companies’ third-party content, and you want to ensure that isn’t blocked by tracker-blockers. This may be more work than implementing DNT just on your own, because you’ll need to work around those third parties’ privacy issues. Nevertheless, some companies have found ways to do this.

4. Because your website has users in the EU and you are aware of regulatory developments there in the field of data protection. You see adoption of DNT as a partial response to the requirements of the General Data Protection Regulation.

5. Because you’d like to be able to tell your users that you’re doing it.

Websites are always reliant on third-party services to some extent, and it is essential to ensure that these companies are capable of providing their services in a manner consistent with the DNT policy. 

----------------------
[1] We believe that DNT is currently the only meaningful privacy opt-out that sites can provide to users. Opt-out systems offered by individual sites or trade associations like the Digital Advertising Alliance and the Network Advertising Initiative are fundamentally flawed. Such systems generally operate on the basis of opt-out cookies whereas privacy- conscious users either block third-party cookies entirely or else clear them regularly. Secondly, and more importantly, it is unrealistic to expect users to explore the privacy options of individual sites, many of which will only be visited rarely and briefly.

## 2. Unique Identifiers and DNT Users
At the core of Do Not Track is the elimination of unique identifiers which allow the compilation of an individual’s browser history over time. This is required as the baseline for all visitors who have set their browser to transmit a DNT:1 header. 

Unique identifiers include high entropy cookies, device fingerprints, supercookies and all similar methods. Cookies can be set only if they are not unique (and thus [low entropy](https://www.eff.org/deeplinks/2010/01/primer-information-theory-and-privacy)), as would be the case for a cookie setting a language preference.

Unique identifiers must not be collected from DNT:1 users who are browsing the sites as guests. If a DNT:1 User logs in then unique identifiers should be discarded and the session token used to identify the user. There should be no logging of user activity unless clear explicit consent has been given for tracking at login (see section 9).

### Example: Database schema for a service that honors DNT:1 for logged in users
Here are some typical database tables and log structures for handling authentication for DNT:1 users, without the need for an opt-back-in exception.

Session table for performing authentication:

| User ID | (Cookie) Session ID   | Expiry    |
| -------|---------------------- | ---------- |
| jane7  | 28bc348ee23681f0912347 | 2018-09-09 |
| juan9  | 20af19d410934120984723 | 2018-11-11 |
| ...    | ...                   | ...       |

Fig. 1

Request table (or server logs)

|URL path                |Time            |Browser            |Result|
|------------------------|----------------|-------------------|------|
|/login                  |2017-07-10 15:31|Firefox 51+ Windows|200 OK|
|/user_profile           |2017-07-10 15:32|Firefox 51+ Windows|200 OK|
|/stories/0293801231290  |2017-07-10 15:34|Firefox 51+ Windows|200 OK|
|/user_profile/add_friend|2017-07-10 15:37|Firefox 51+ Windows|200 OK|

Fig. 2

Note that server logs and database tables do not contain any entries or rows that connect the requested pages to the user ID or their unique authentication cookies. Those cookies are inspected as the server handles the request to ensure the user is correctly logged in, but are not subsequently logged. Note also that the session expiry timestamps are rounded to the nearest day, to prevent login time from being an implicit column for joining these two tables.

Most methods for generating and passing [CSRF](https://en.wikipedia.org/wiki/Cross-site_request_forgery#Prevention) tokens are compatible with this design. If any CSRF secrets function as unique user identifiers, they should not be logged in a way that can be matched or joined on the server side with the pages the user is requesting.

For instance if CSRF tokens are generated via an [HMAC](https://www.icir.org/vern/cs161-sp17/notes/CSRF_Paper.pdf) of a secret, keep a table like this:

Session table for performing authentication

|User ID|CSRF secret          |Expiry    |
|-------|---------------------|----------|
|jane7  |ff47324291121031239ab|2018-09-09|
|juan9  |093deb20390ad1091432 |2018-11-11|
|...    |...                  |...       |

Fig. 3

## 3. How to Limit Web Server Logs
The limitations on log retention are crucial so it is worth quoting the policy in full:
> 2. LOG RETENTION:
> a. Logs with DNT Users' identifiers removed (but including IP addresses and User Agent strings) may be retained for a period of 10 days or less, unless an Exception (below) applies. This period of time balances privacy concerns with the need to ensure that log processing systems have time to operate; that operations engineers have time to monitor and fix technical and performance problems; and that security and data aggregation systems have time to operate.
>
> b. These logs will not be used for any other purposes.

These are some basic examples of server configurations for Apache and Nginx to help you get started implementing server-side DNT. Every system has its own unique characteristics so please treat these samples as merely illustrative of an approach that will need adaptation to work correctly. EFF is not responsible for any problems which may arise.

Note that to test the following configuration samples, you can toggle DNT on/off through Firefox and Google Chrome.

### Apache
There are several ways to configure Apache so that requests with DNT:1 are not logged for more than 10 days. All methods listed require Apache version >= 2.0.
 
For the following examples, the code provided belongs in the relevant VirtualHost directive in your site’s conf file (usually `/etc/apache2/sites-available/example.com.conf`). 

__Approach #1:__ Simply do not log any requests from users who’ve set DNT:1. Here’s a configuration snippet from [Arvind Narayanan](http://randomwalker.info/) and [Jonathan Mayer](https://jonathanmayer.org/):
```
SetEnvIfNoCase DNT 1 DO_NOT_TRACK
CustomLog ${APACHE_LOG_DIR}/access.log combined env=!DO_NOT_TRACK
```
Note: the above should replace the default `CustomLog` directive that usually looks something like:
```
CustomLog ${APACHE_LOG_DIR}/access.log combined
```
Note: “`combined`” just refers to the logging format you want to use. If it said something else before (e.g. “`common`”), you can use that instead. “`combined`” is a popular choice that logs the most information by default.

__Approach #2:__ log requests from users with DNT:1 separately and configure logrotate to delete these after they’re 10 days old.

For the sake of simplifying logrotate’s configuration later on, you’ll need to create the following directory: `/var/log/apache2/dnt/` as a first step.

Note: it’s important that the following Apache code snippet is placed above other `CustomLog` lines which might catch those requests.
```
SetEnvIfNoCase DNT 1 DO_NOT_TRACK 
CustomLog ${APACHE_LOG_DIR}/dnt-access.log combined env=DO_NOT_TRACK
<the rest of your logging configuration goes here>
```
See the Logrotate section below for the rest of the configuration instructions.

__Approach #3:__ If you don’t want to log requests with DNT:1 separately, you can modify the format of the logs via LogFormat to capture the DNT field at the end of the log file. Then once a day run a script that checks the logs from 10 days ago and strips out entries if they end in 1.


### Nginx

Note that in the following examples, we assume that you are using the default `/var/log/nginx/` directory for your Nginx logs. If you keep them somewhere else, adjust accordingly.

__Approach #1:__ Simply do not log requests from users who’ve set DNT:1. 

In your site’s config file (usually `/etc/nginx/site-available/example.com`), add the following above the `server` directive:
```
map $http_dnt $loggable {
    		1 0;
    		default 1;
}
```
And add the following inside the `server` directive:
```
access_log /var/log/nginx/access.log combined if=$loggable;
```

__Approach #2:__ Log those requests separately.

First, create this  directory: `/var/log/nginx/dnt/`

In your site’s config file (usually `/etc/nginx/site-available/example.com`), add the following above the `server` directive:
```
map $http_dnt $loggable {
    		1 0;
    		default 1;
}
```
And add the following inside the `server` directive:
```
access_log /var/log/nginx/access.log combined if=$loggable;
access_log /var/log/nginx/dnt/access.log combined if=$http_dnt;
```
See the Logrotate section below for the rest of the configuration instructions.

### Logrotate

To configure logrotate to delete these files every 10 days, we’ll need to add a separate logrotate config for the new DNT log we’ve created. To do that you’ll need to find your server’s logrotate config file (default for Apache 2 is: `/etc/logrotate.d/apache2`, and Nginx is `/etc/logrotate.d/nginx`) to include the following:
```
/var/log/nginx/dnt/access.log {
		maxage 10
		<whatever else you normally have in here>
}
```
Make sure to change `/var/log/nginx/dnt/access.log` to wherever the DNT logs are stored.

### What about other web servers?
Know how to accomplish this for other webservers? Help us by submitting a PR in this repo!


## 4. Ensure Compliance by Third Parties
There are three methods to ensure compliance by your third parties:
	
1. Policy adoption by the third party: they can adopt it by checking they meet the requirements for compliance and posting the policy at https://example-domain.com/.well-known/dnt-policy.txt

2. Contract: You can specify in your service contracts that suppliers must meet the standards set out in the DNT policy for users with DNT enabled.

3. Technical measures: If the third party cannot or will not abide by the terms of the DNT policy, then technical measures should be taken to reduce or stop the flow of DNT users data to them. 

## 5. How to Embed Content from Third Parties
Third-party services are commonly used to serve images, video and audio. By default, these sites are passed the IP address and user agent data of the user and can set their own cookies. Similar considerations apply to social sharing buttons such as Facebook’s "like" button and Twitter’s "tweet this". We understand that these are elements that webmasters may want to use to encourage the further distribution of their content and so as to measure how successful a given piece is.

#### a. [Embed.ly](http://embed.ly/)
Embed.ly is a tool for embedding media on a website and supports more than four hundred providers. These media elements can be presented in a clean form with user control over the display specifications. Embed.ly supports DNT wrappers for embedded widgets. This means that users are required to click in order to activate the embed. Normally the user is tracked as soon as the page is loaded. Embedly supports most common widgets including YouTube, Vimeo, Sound Cloud, Twitter etc.

Documentation as to how to generate hosted embeds can be found here:
http://docs.embed.ly/docs/oembed

For now, the DNT:1 code must be added manually.

The resulting DNT poster looks like this:

![Embedly DNT Image](/assets/embedly-dnt.png)
Fig 4.

#### b. Shariff
Shariff is the successor of [Two Clicks for Data Protection](https://panzi.github.io/SocialSharePrivacy/), its purpose is to protect user privacy from tracking through social media buttons. This project is maintained by the German magazines c’t and Heise. The [project page](https://github.com/heiseonline/shariff) states:

>“Facebook, Google+ and Twitter supply official sharing code snippets which quietly siphon personal data from all page visitors. Shariff enables visitors to see how popular your page is on Facebook and share your content with others without needless data leaks.” 

Integrations are available for many third-party systems such as Word Press, Drupal and Joomla.

## 6. Designing anonymized data structures

The DNT policy contains an exception for the generation of anonymised data structures, this is a tricky area so it's worth quoting the relevant text: 
> 4. AGGREGATION:
>
>  a. We may retain and share anonymized datasets, such as aggregate records of readership patterns; statistical models of user behavior; graphs of system variables; data structures to count active users on monthly or yearly bases; database tables mapping authentication cookies to logged in accounts; non-unique data structures constructed within browsers for tasks such as ad frequency capping or conversion tracking; or logs with truncated and/or encrypted IP addresses and simplified User Agent strings.
>
> b. "Anonymized" means we have conducted risk mitigation to ensure that the dataset, plus any additional information that is in our possession or likely to be available to us, does not allow the reconstruction of reading habits, online or offline activity of groups of fewer than 5000 individuals or devices. 
>
> c. If we generate anonymized datasets under this exception we will publicly document our anonymization methods in sufficient detail to allow outside experts to evaluate the effectiveness of those methods.

### A. Common Anonymized Data Structures
There are lots of cases where sites need to analyze, process and share data about site usage, user behavior, ad clicks, etc., for numerous purposes. The DNT policy allows such collection, analysis, and sharing, provided the data is properly anonymized. Below we offer some sample data structures to address popular requirements. For some uses, anonymous processing will meet your needs for all users. In other cases, you may wish to keep full data about non-DNT users, and rely on the anonymous analysis for the users who have opted out.

#### Example 1: Aggregating Readership Patterns
If you want to know how popular various content URLs on a site are, a naive implementation might be to use an analytics solution that records every (`user`, `URL`, `timestamp`) record indefinitely. Unfortunately, this would violate the DNT policy if some of the users are DNT: 1 users.

An anonymized alternative is to build a database table like the following:


| URL path | Time Period | Visit count | DNT visit count |
| -------- | ----------- | --- | --- |
| /blog/post/xyz | 2017-03 | 1,328,422 | 111,121 |
| /blog/post/xyz | 2017-04 | 43,530 | 5,893 |
| /blog/post/xyz | ... | ... | ... |
| /features/march/abc | 2017-03 | 2,330,325 | 345,682 |
| /features/march/abc | 2017-04 | 876,983 | 17,439 |
| /features/march/abc | ... | ... | .... |

Fig. 5

By querying this table, you can measure the popularity of URLs even if you obfuscate or delete regular log lines or (`user`, `URL`, `timestamp`) records for DNT: 1 users.  

Note that if you do not have a record of which of your users have enabled DNT, there is no concern about identifying the reading habits of a group of 5,000 or fewer people. If you have a separate table mapping your user IDs to DNT status, you need to ensure that you have at least 5,000 DNT users before including the “DNT visitor count” column in the table above.

#### Example 2: Conversion Tracking

Supposing an adtech platform _P_ wishes to be able to provide evidence to firms _F_ that buy ads on its platform that a DNT user _u_ who purchased an advertised product saw some number of ads in the campaign.

The third-party advertising widget includes some JavaScript which stores an entry in client-side [localStorage](https://www.w3schools.com/html/html5_webstorage.asp) each time an ad from a campaign is displayed. This may be a simple count of impressions, or (if the information stays client-side or is otherwise dissociated from the server’s ability to log the user’s identity) it could include information like the first-party domain the impression was served on.

_P_ also provides F with code to include on their checkout page. Once a transaction purchasing a product by _u_ has been confirmed, _P_’s third-party JavaScript can read information out of `localStorage` to tell _F_ how many times _u_ saw an ad from the campaign, and bill for conversions accordingly.

Note that _P_ should not itself learn, or tell _F_ which specific sites or pages _u_ saw the ad on. But _P_ can build a separate anonymous data structure using a technique like RAPPOR (see below) to provide a report to _F_ of aggregate conversion rates by site or URL.

#### Example 3: Fast Anonymous Frequency Capping with Contextual Persistence

The anonymized conversion tracking algorithm described above would also work for frequency capping and some forms of anonymous behaviourally relevant advertizing. However, it would require JavaScript from a third party to load, run, and query `localStorage` before inserting the ad image into the DOM. For some advertising platforms, this chain of JavaScript events is too slow and so other solutions may be useful for frequency capping.

One fast solution is to sort users into buckets using a low entropy cookie. The maximum number of states this cookie can have is one five-thousandth of the advertiser’s estimate active user count during the lifetime of the cookie.  For instance, the advertiser might pick a 14 day cookie, and might see 100,000,000 fortnightly active users.  They could then set a 14 day `Impressions` cookie, which groups users into 20,000 different equally-likely buckets.

There are many ways to use those buckets for frequency capping, but one example would be: build a server-side array of 20,000 ads, with campaigns included in proportion to their current bid levels or other relevant business logic. Typically a given ad or campaign would be included in the array many times, but if an ad is to be capped at N impressions, never insert more than _N_ copies of the ad near each other in the array. When a DNT user requests third-party content from the advertiser, use the cookie their browser sends to select the ad that will be served to them, and increment the cookie index by 1 (or if you want to hide the structure of this array from your competitors, use a set of 20,000 valid random numbers in a much larger space of invalid ones, and give the user the next valid cookie number with each request).

If a user does not have an `Impressions` cookie, initialize it to a value which is randomly chosen from a range that is contextually relevant to the site in question. 

It is not safe for the advertiser to keep logs of the `Impressions` cookie joined with IP addresses or web server logs (because it stores contextual information across requests), but the advertiser can build separate tables to do aggregated analytics on `Impressions` by first party site or URL.

### B. A Toolbox of Technical Resources for Designing Anonymized Data Structures

#### I. A Cautionary Tale...

The EFF DNT policy allows retention, use, and sharing of anonymized data, provided appropriate risk assessment and mitigation is performed. Anonymization is hard, and sites intending to generate their own custom data sets should make sure that they are up-to-date on developments in the field of re-identification. Even when the data set will be used only internally, its generation should be modeled on the worst-case scenario that an inadvertent or unauthorised leakage has taken place. Re-identification is an extremely dynamic field of research and there have been many successful re-identifications of publicly released data in the last two decades, resulting in privacy loss for the users involved and negative exposure for the organization. Major incidents include:

* In 1997 an insurance company in Massachusetts released a database with information about hospital visits by state employees along with any resulting prescriptions. Obviously identifying information such as names, phone numbers, addresses and social security numbers were removed. Latanya Sweeney, a privacy researcher, bought a voter record database for $20 and cross referenced the two data sets. She was easily able to identify the State Governor’s own records based on a combination of zip code, date of birth and gender. She showed that 87% of US residents could be identified in this manner, demonstrating that [Personally Identifiable Information is not a clearly defined category.](https://www.eff.org/deeplinks/2009/09/what-information-personally-identifiable)
* In 2006 Netflix released a hundred million movie ratings by nearly half a million of its subscribers. User names were removed and replaced by numerical identifiers that were consistent across rating records. Arvind Narayanan, a researcher, aggregated ratings to user numbers and then matched them against public reviews from the Internet Movie Database. This simple technique allowed him to identify several users and unlock their viewing history.
* In 2014 a Freedom of Information Act request forced the New York City Taxi and Limousine Commission to release a database of every yellow taxi ride in 2013. The de-identification process had multiple [flaws](https://tech.vijayp.ca/of-taxis-and-rainbows-f6bc289679a1), one of which was that exact pickup and dropoff data was provided in the form of GPS coordinates. [An examination of rides](https://research.neustar.biz/2014/09/15/riding-with-the-stars-passenger-privacy-in-the-nyc-taxicab-dataset/) originating at a strip club in Manhattan revealed that in some cases the drop off points were locations with few or even one residential address. That address, in turn, allowed the identification of the user through a Spokeo search.
* In 2016, the Australian Federal Government released a dataset of patient medical histories from Medicare, Australia’s universal public healthcare system. The anonymization involved the use of encrypted identifiers for patients and providers in the system, but researchers quickly found that, at least for the providers, [the encryption was flawed and easily broken.](https://pursuit.unimelb.edu.au/articles/understanding-the-maths-is-crucial-for-protecting-privacy)

These cases underline the difficulty of reliable de-identification. First, other datasets may already, or in the future, be available to be cross-referenced, transforming what looks like harmless information into an identifier. Second, those datasets may never have been intended to reach the public (as was the case with the Taxi data). Thirdly, the party trying to re-identify the data may bring specific knowledge about the targeted individual to bear in querying the data. This is easy to imagine in the case of a motivated attacker, be it a private investigator or a troll.

#### II. Tailor the Approach to the Data and the Goal
The appropriate method for de-identifying a database varies according to its purpose. This will determine whether privacy protection can be integrated during collection (by adding noise) or only later (through noise or the selective distortion of the data).

The table in Fig. 5 above provides an ideal model for what the data should look like if it meets the 4(b) requirement of each tuple containing a minimum of 5000 users. Where the number of users is insufficient, the field is left blank. IP addresses and unique identifiers have been removed and timestamp data is generalised. This process of suppression and generalization of records enables the creation of batches of more than 5000. There are tools for cleaning data sets such as [ARX](http://arx.deidentifier.org/overview/). 

Other types of data, such as the relative popularity of a set of pages, do not require knowledge of individual behaviour -- the insights sought derive from what the group is doing on aggregate. This data can be collected more safely using methods and tools that provide [Differential Privacy](https://blog.cryptographyengineering.com/2016/06/15/what-is-differential-privacy/). Differential Privacy is achieved by introducing noise into the data collection or analysis process. A classic example is to have a portion of participants respond to a query not based on the truth but on a coin flip, randomizing the responses. Over a large group of respondents, analysts can make adjustments to compensate for this noise and retain the utility of the results without exposing the individual participants.

Differential Privacy is a field in rapid development and anyone intent on implementing such techniques should pay close attention to the most recent developments. Companies such as Google, Apple, Microsoft and Uber are all using Differential Privacy and their implementations are feeding research and criticism. The most tested tool available is currently [RAPPOR](https://github.com/google/rappor), developed by Google and described [here](https://www.chromium.org/developers/design-documents/rappor). In October 2017 the team behind it released a paper outlining an updated model and implementation, [RAPPOR 2.0.](https://arxiv.org/abs/1710.00901)

#### III. Document your Methodology

There is [no silver bullet re-identification.](http://randomwalker.info/publications/no-silver-bullet-de-identification.pdf) When generating a dataset, you should research the appropriate methodology and then ensure that you have the internal resources to implement it correctly. If not, seek expert external assistance. 

Our policy requires that, if information about DNT users is compiled into data sets, then the processor must both carry out risk assessment and minimization, and document the method used. If you intend to release the data, consider having an external review by a specialist company such as [NCC.](https://www.nccgroup.trust/)


## 7. Analytics
__Piwik__

[Piwik](https://piwik.org/) supports Do Not Track by default and offers an option to anonymize IP addresses. Piwik's creator, Matthieu Aubry, [explains](https://forum.piwik.org/t/piwik-neither-respects-do-not-track-nor-explicit-opt-out/21831/2):
“Requests are sent to the Piwik tracking API which decides whether to drop them off if DoNotTrack is enabled. So it is expected the requests are sent, but they should not be tracked and not appear in your reports.”

A Piwik [plugin](https://wordpress.org/plugins/wp-piwik/#description) is also available for WordPress.

__Google Analytics__

Google does not offer any options to make Google Analytics compliant with DNT. While GA does offer the option to delete the final octet of the IP address, this is not sufficient as the number of potential users/devices is limited and well below our minimum bucket requirement of 5000 users.

GA can be made compliant by preventing it from accessing any data about DNT:1 users. [Outfox](http://www.outfox.com/do-not-track-for-google-analytics/) offer a free plugin to enable DNT support which we have successfully tested (Outfox provides no guarantee of technical support for this product).


## 8. Handling OCSP In a Privacy Preserving Manner
To implement DNT you will need to support HTTPS and acquire a signed public key certificate from a Certification Authority. Checking the validity of your site’s TLS certificate requires that users expose their browsing activity to the CA operating the OCSP server. To avoid this, sites should either [implement OCSP stapling](https://www.digitalocean.com/community/tutorials/how-to-configure-ocsp-stapling-on-apache-and-nginx) or use a CA that is DNT-compliant, i.e. does not log requests from DNT:1 users, such as [Let’s Encrypt](https://letsencrypt.org/). 

Ask your SSL Provider to post a DNT-compliant policy. If they do agree to do so, let us know and we will add them as a resource in this document. 

## 9. DNT-Compliant Cloud Service Providers

Many sites use third-party Content Distribution Networks (CDNs) and services that mitigate against Distributed Denial of Service attacks (DDOS). A lot of companies operate in this space and it is impossible for us to check them all. The key questions here are whether the service is setting unique cookies of its own (in addition to those of the website) and if they independently generate logs. Often this will be a matter of how the user configures the service, see [Fastly's documentation](https://docs.fastly.com/guides/compliance/security-program#customer-and-end-user-data-management) for example. Keeping logs is not in itself a bar to compliance, but the logs cannot be retained more than ten days. Usually, these companies are setting the same policy for all users.


| Provider | Logs | Cookies | Compliant |
| -------- | ---- | ------- | --------- |
| Fastly | No | No | __Yes__ |
| MaxCDN | No | No | __Yes__ |
| Cloudshare | No | Yes | No |
|Amazon Cloudfront | User defined | Yes | No |
| Akamai | Yes | Unspecified | No |
| Deflect | No | No | __Yes__ |

For websites involved with independent media or non-profit work, [Deflect](https://deflect.ca/) provides free DDOS mitigation with strong privacy features. They do not set cookies unless they first identify suspicious activity. Customers can choose in their settings whether they want to collect visitor logs or not. 

## 10. How to get Consent Permission for Additional Data Retention

Sometimes companies will want to build features that inherently require recording of user activities. More often they have business models that inherently require tracking. Additional tracking is permitted under the EFF DNT Policy, _but only if the user has given clear non-confused consent._ Sites should also strive to ensure that features and modes of operation that require tracking are optional, and that reading or using the site anonymously remains possible.

Medium is a great example of implementing EFF DNT by this path.  On Medium, being logged out and sending DNT: 1 is “stealth mode”. If the user logs in, they consent to retention of their data:

![Medium DNT Image](/assets/medium_dnt_lang.png)

In addition to deploying that clear notice to users at log-in time, Medium also did some work to improve the quality of the not-logged-in experience on their site, and disabled a number of third-party embeds for DNT: 1 users who were not logged in. For many companies, that’s all you’ll need to do to be DNT-compliant.

__Setting Tk Headers for W3C__
If you collect consent to track like this at log-in time (or some other time), your web application should send a [Tk](https://www.w3.org/2011/tracking-protection/drafts/tracking-dnt.html#responding): [C](https://www.w3.org/2011/tracking-protection/drafts/tracking-dnt.html#TSV-C) header to let the browser know, we will follow the indications of the [Tracking Preference Expression](https://w3c.github.io/dnt/drafts/tracking-dnt.html#TSV-C) specification from the W3C. 

This ‘C’ value will appear in the user interface to indicate that your domain is tracking the user with their consent, and may be revoked at any time by the user. 


## 11. Other Exceptions to the policy

Where the user deliberately engages in any form of transaction, the site owner may collect all the information necessary. Examples include:
* A purchase; this will require buyer’s name, address etc and the retention of order related information until such time as it is clear that the transaction will not be disputed.
* A comment; comments or posts to forums will usually require the retention of the post, the name of the user etc.
* Where a user clicks on an ad creating a billable encounter the information needed to audit this can be retained.
* Where the circumstances of a request give rise to the suspicion that it may be connected to spam, fraud or a security threat, these requests will be treated as being outside the policy and the site may use cookies and retain the log data as necessary.
* If necessary for debugging or resolving technical issues on the site then logs can be retained for longer than ten days. As soon as the problems have been resolved however this data should be deleted and this exception is to be understood as being allowed only in exceptional circumstances; it cannot legitimate a generalised practice of log retention under a pretext of general technical necessity.
* Inadvertent errors made in good faith will not be treated as violations of the policy.

## 12. How to Assert DNT Compliance 

Now that you have implemented all of the privacy requirements listed in our Do Not Track policy document [https://github.com/EFForg/dnt-policy/blob/master/dnt-policy-1.0.txt] you can let the world know that you support Do Not Track and web privacy! The first step is to post the latest version of the Do Not Track policy to any domain you control which is now in compliance. The document should be posted at `your-domain.tld/.well-known/dnt-policy.txt`. Do not change the text of the document in any way (for example, don’t change example-domain.com to your domain). You can assert that you have done this correctly with the following command:
`curl your-domain.tld/.well-known/dnt-policy.txt | shasum`

comparing the hash to one of the acceptable hashes listed in https://github.com/EFForg/dnt-policy/blob/master/dnt-policies.json 

__W3C TSR__
A website can also signal compliance within the framework of the W3C Tracking Preference Expression (DNT) Framework. This requires posting [a Tracking Status Representation](https://w3c.github.io/dnt/drafts/tracking-dnt.html#status-representation) (TSR), which should contain a tracking status and a compliance property. A link to a hash of the EFF DNT policy can be posted here. A site may want to do this if it is supporting multiple compliance regimes.

__First vs. Third Party?__
The DNT policy applies on a per domain basis. This means that you can select which resources and services you want it to apply to so long as you serve them off different subdomains. Thus you can decide that you don’t want it to apply to your content creation activity at https://www.exceptional-writing.com but are happy to support DNT when serving ads to other fiction sites from ads.exceptional-writing.com, i.e. when providing a third-party resource to another site. Adoption of the policy on one domain does not bleed into the other and the policy should always be posted on the relevant URI, e.g. https://ads.exceptional-writing.com/.well-known/dnt-policy.txt

__Let the World Know!__
Now that you have posted the DNT compliance document please email dnt-policy@eff.org to let us know so that we can add you to our list of compliant sites. [https://www.eff.org/files/effdntlist.txt]

### Reassertion of compliance

Do not forget to periodically reassess whether the domains which you have posted the DNT policy to are still in compliance with the document. Such a review should take place at least annually but also whenever the site undergoes significant reengineering. You can reassert by either:
a) Tweeting from your from a known official site account that you are still following the policy.
b) By updating a page, whether the site's privacy page or another linked from it, to state that you last carried out a review on 'x' date and are in compliance.

A domain which falsely declares itself compliant with the DNT policy document could face legal trouble.

### Regulatory Regimes
 __California AB 370__
 
Regardless of where your website is located, this [law](https://leginfo.legislature.ca.gov/faces/billNavClient.xhtml?bill_id=201320140AB370) modifies the California Online Privacy Protection Act to require that a site discloses how they respond to the Do Not Track signal. Some sites will try and filter their Californian users but it is most likely easier to provide the same information to all users. Document your DNT support in your privacy policy.


 __General Data Protection Regulation (GDPR) and ePrivacy Regulation__
 
Companies established within the EU, or currently processing information within the EU, must meet the requirements of the 1995 Data Protection Directive and the 2009 Cookie Directive. Next year this framework will be superceded by the [General Data Protection Regulation](http://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32016R0679&from=EN).

_Scope and Reach_
  
From May 2018, if a website processes data of users within the European Union, even if it is not established there, it is required to comply with the GDPR and also the forthcoming ePrivacy regulation (as yet unfinalized). The former sets out the general rights of users, the latter sets out specific restrictions regarding the use of cookies and other similar data stored on and retrieved from an end user’s device. EU Data Protection law states that sites must post a privacy notice making users available of their rights and sets out a limited series of grounds for processing users’ personal information. Personal information is defined widely and includes IP addresses and cookies:

>‘personal data’ means any information relating to an identified or identifiable natural person (‘data subject’); an identifiable natural person is one who can be identified, directly or indirectly, in particular by reference to an identifier such as a name, an identification number, location data, an online identifier

EFF’s DNT Policy sets out a framework which, if followed, will make sites partially but not fully compliant. The crucial element is that the limitations imposed by the policy on the data of DNT users must be applied to all users. 

_Limitations on Storage of Data_

The GDPR, in contrast to the DNT policy, does not define global storage durations within which a controller can retain personal data. Data controllers are required to provide users with notice of either a) 'the period for which the personal data will be stored' or b) 'the criteria used to determine that period' both at the time of collection (Article 13), and in the case of a subject access request (Article 15). In addition, any controller seeking certification of Binding Corporate Rules (A.47) for the transfer of personal data to third countries is also required to specify 'limited storage periods' (Article 47(2)(d)).  

As a general matter their practices are subject to principles of 'data minimization' and 'storage limitation' as set out in Recital 39: 

>The personal data should be adequate, relevant and limited to what is necessary for the purposes for which they are processed. This requires, in particular, ensuring that the period for which the personal data are stored is limited to a strict minimum. Personal data should be processed only if the purpose of the processing could not reasonably be fulfilled by other means.

Article 5 requires that personal data is "kept in a form which permits identification of data subjects for no longer than is necessary for the purposes for which the personal data are processed", an approach echoed in Article 25 which mandates "Data protection by design and by default."  

The GDPR creates new obligations and liabilities, which together form a sound business argument for minimizing data collection and retention. Any data held by a controller also creates data security obligations, and is also subject to the open-ended ongoing obligation to provide the data to subjects under access requests (without charge to the subject).  

If you comply with the EFF's DNT policy, and retain standard logs for no more than ten days for all users, these burdens will be much reduced and your practices arguably meet the tests contained in the regulation. Where information is retained under one of the exceptions, the website operator should set out the periods normally required.

_The Right to Object to Processing for Marketing Purposes_

Article 21 sets out "the right to object" to processing. Paragraph 2 specified that this includes  marketing purposes and paragraph 5 states that "the data subject may exercise his or her right to object by automated means using technical specifications." DNT is not explicitly referenced by the GDPR, but is for now the only automated means that can communicate a user’s opt-out. Later this year the Article 29 Working Group, which brings together European Data Protection authorities to formulate advice on the interpretation of the law, will issue an opinion on this aspect of the Regulation.

The ePrivacy Regulation is a "child" of the GDPR and will impose additional requirements in terms of user privacy and tracking of various types, including the use of cookies and device fingerprinting. This law is still under negotiation but it will narrow the conditions in which cookies are used and bolster user privacy at the browser level.

_Consent & Purpose_

The GDPR requires that processing of personal data only take place with a legal basis. Consent is the most commonly discussed legal basis but there are alternative grounds such as contractual necessity, compliance obligations and the ‘legitimate interest’ of the processor (Article 6). 

Conditions for valid consent are set out in Article 7, and it is defined in Article 4 as an indication which is “freely given, specific, informed and unambiguous”. The meaning of these terms is unpacked in Recitals 32, 42 and 43 clarify the effect of these criteria.

- For consent to be free, it cannot require that access to a service is made contingent on consenting to uses of the data that have nothing to do with the provision of the service.
- In order to be “specific”, the purpose of the collection and processing must be communicated. It is not possible to bundle permission for a range of uses under one block of legal boilerplate.  
- The user must be “informed”, which means that the identity of the controller and the purpose of the processing be made known to them. 
- “Unambiguous” means that “a statement or … a clear affirmative action” is required to grant consent, “Silence, pre-ticked boxes or inactivity should therefore not constitute consent.“

Recital 39 makes clear that these conditions must be satisfied at time of collection, "In particular, the specific purposes for which personal data are processed should be explicit and legitimate and determined at the time of the collection of the personal data.”

EFF is working with groups in the EU to produce further GDPR compliance guidance for publication in 2018.

__All Done?__
Congratulations! You are an official supporter of EFF’s Do Not Track standard! Now DNT supporting content blockers such as Privacy Badger, Disconnect, AdNauseam and others will not block any content coming from your DNT-compliant domains.
