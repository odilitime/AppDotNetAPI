AppDotNetAPI (AKA AltAPI)
============

An alternative implementation of the App.net API implemented as an App.net client. This is an attempt to make the App.net API an open standard.

This is a piece of software that you can run to emulate the App.net API that all the existing App.net apps could connect to (if we can convince 3rd party app developers to include a configurable API/OAuth root in their App.net software).

This way we can keep the community and software intact.

Compatibility is goal #1.

#Upstream integration

It can work as a back up to the main App.net. So while App.net is running everything could feel like one network. In the worst-case scenario where App.net goes away forever, AltAPI would serve as a backup for preserving the community and allowing people to use former App.net apps, now with altapi.

A local AltAPI account can be linked with an App.net account. All locally created posts can be cross posted to App.net. All App.net created posts can be streaming into our data source and out to clients, ensuring seemless two way posting between the networks on a per user basis.

An API server operator can decide whether to allow non-linked App.net accounts to be able to post or not. If not allowed, there would be a complete two-way App.net relay and backup; this is because an App.net account posting to that AltAPI instance would be linked to a local AltAPI account, which means that the data would propagate through the AltAPI network and be preserved. Otherwise, if nonlinked App.net accounts are allowed to post then the server operator will end up with content only available on their node.

This client/server relationship actually allows an API server whether it's App.net or other API compliant servers to link up in a hub-spoke topology and distribute the network across additional resources, though work and planning will need to be put into distributed authentication and authorization.

Users would authorize AltAPI with their App.net accounts, just like they would any other App.net app. We envision users selecting from a list of AltAPI nodes before authorizing with it.

The authorization flow itself would happen via OAuth proxy. This means that OAuth requests propagate up nodes until reaching the top node (usually App.net; otherwise the next connected AltAPI node would take its place). At each successive node, a token is returned, then the request continues upstream. When the subsequent node returns a token, that remote upstream token is mapped to the token that the requesting node returned--the local token.

As well, one could set up their own standalone social network and embrace all the App.net clients that support setting an alternative API root.

#Components

## Daemons
We'd need a web daemon for serving web requests.
Then a background daemon for upstream streaming (connects to App.net using app streaming), maintenance or background processing.
And then a websocket daemon for user streaming.

The web and background daemon will be written in NodeJS as they'll need to share code for creating posts. 

The websocket daemon will be written in Go since it'll just be reading and no writing.


##OAuth and sign up
We will need to implement our own OAuth server and sign up process, since we don't want users using their real App.net username and password.

##Data flow

###Dialects (dialect.*.js)
The server will support different "dialects" of the API. The first one will be "appdotnet_official" which will be the 100% compatible implementation. These will create ExpressJS mountpoints in the NodeJS web server.

New dialects will be able to extend the App.net API without breaking existing clients. A server is able to run multiple dialects at once.

###Dispatcher (dispatcher.js)
This mainly translates internal API calls to DataAccess Chain. Dispatcher talks to the configured "cache". The upstream app stream and the DataAccess chain will feed data through the dispatcher for transformation and future storage in the DataStore.

###DataAccess Chain (dataaccess.*.js)
The DataAccess chain is a list of DataAccess objects. Each objects in the chain will have the same exact API. If a query is not in current DataAccess layer, it will send the request to the next DataAccess layer in the chain until the request is served. Writes can cascade up too in case there are mulitple upstream providers to post to each one individually.

####DataAccess: Cache
This is just a pass-through at the moment (dataaccess.base.js). Eventually hand-crafted modules will be created. These modules will have more performant data structures than the data store can provide and can be used to optimize slower API paths.

####DataAccess: Data store
This contains the models for the caminte ORM (dataaccess.caminte.js) and the API to get and set data in the Data store. 

This will be segmented into 4 backing stores: Auth, Token, Rate limiting, and Data. You will be able to configure each backing store to use a different storage backend (memory, SQLite, Redis, MySQL). 

This layer will be responsible for managing expiration and size of the data set. 

####DataAccess: Upstream Proxy
Requests will be turned into requests that can be proxied from an upstream server. 

###Upstream streaming
If configured, the NodeJS web server will create an app token with an upstream network (App.net at the moment) and use app streaming to receive network updates and populate its cache/data store through the dispatcher.

##Current Performance
I'm finding (with taking in account network latency) that the performance of the NodeJS web server is 10-20 faster than the official App.net API (Using memory or Redis data stores). This is likely due to the small datasets I'm testing with. Only time will tell with the larger datasets if the performance will hold steady.

##Potential Scalability
Using Ohe's lightPoll model; I believe with a message queue, we can have multiple web worker nodes (only one master node with upstream connection). This will allow for additional scalability if you wish to do a large scale deployment or just use all the cpu cores in one server.

#I'm an App.net 3rd party developer how can I get my app ready for an alternative API server?
We just need you to have the root of the API request to be configurable.

So if you have "`https://api.App.net/`" or "`https://alpha-api.App.net/stream/0/`" are your API root right now, you just need an UI for your users to be able to enter an alternate root. This will allows users to select what API to connect to and use.

We'll also need to allow your OAuth endpoints to be configurable, as the users will have different usernames and passwords than their App.net counter parts.

We're talking about implementing a JSON data source that will provide a directory of all the available AltAPI servers. This will be for the 3rd party app devs that really want a really nice UI for selecting an AltAPI server.

# Possible network configurations
##ADNFuture
We will be creating a new spoke from App.net. Other people that want to run servers can spoke off our hub implementation. This allows App.net to go down and for us to keep the network going. We may introduce a peer to peer mode under our hub to help distribute load as well.

App.net => ADNFuture => Multiple providers

##Local cache
You can run your own local cache to speed up your App.net network interactions:

ADNFuture => Your local cache

##Standalone
Start your own social network using this software as the main hub and base. You can allow or deny other spokes from connecting to your hub.

Your Hub => Possible spokes

# Requirements

* Node.js 0.8.xx+

# Installation

1. Copy config.sample.json to config.json

1. [optional] Create a new application on App.net. Note the client_id and client_secret and put in `config.json`. The redirect URI should be /return on the host you're going to use for AltAPI, e.g., <http://localhost:7070/return>.

1. `npm install` to get node js libraries

1. `node app.js` to run the server

1. Open your browser to <http://localhost:7070/>

# Security
It's best practice to not have clients directly connect to node.js. We recommend use stunnel or nginx to provide an HTTPS connection to ensure data is encrypted while it travels over the Internet.

#Roadmap

##Phase #0 - Public proxy - Complete 
This is an easy target to lay out the base foundation. We just need a webserver. We will just proxy App.net data.

+ posts
  + /posts/1
  + /users/1/posts
  + /users/1/stars
  + /posts/tag/jukebox
  + /posts/stream/global
+ channels/messages
  + /channels/1383
  + /channels/1383/messages
  + /channels/1383/messages/3442494
+ configuration
  + /config
+ text processor
  + /text/process
+ oEmbed
  + /oembed?url=https://posts.App.net/1

##Phase #1 - Public endpoints - In Progress
We have added a data store to the project at this point. We will stream, store and relay App.net data. We're adding a lot of structure here. 

Same endpoints as Phase #0

###Phase #1.1 - Storage structure - Complete
###Phase #1.2 - Proxy uses DataAccess Chain - Complete
###Phase #1.3 - Parameters - In progress
####Paging paramters - In progress
#####Complete:
+ posts
  + /posts/stream/global

#####Incomplete:
- posts
  - /users/1/posts
  - /users/1/stars
  - /posts/tag/jukebox
- channels/messages
  - /channels/1383/messages
  
#####ids parameter endpoints
- posts
  - /posts?ids=
- channels/messages
  - /channels
  - /channels/messages

####channel_types parameter
#####include_* parameter
I don't think any app functionality is harmed by having too much data, so we skip the parameters that are on by default. We will initially always return annotations, stream_marker for now. I've prioritize the following include as I find them to be most harmful to applications:

- posts
  - include_muted
  - include_machine
- channel
  - include_read
  - include_recent_message
  - include_inactive
- messages
  - include_muted
  - include_machine

all other include_* parameters will come at a later date.

##Phase #2 - Authenticated endpoints - In Progress
The goal is to allow you to authorize your official ADN account with the AltAPI. (This is complete) And with the token authorization start building the token-require API endpoints.

We will start developing the write endpoints as well as:

In Progress:

+ post
+ star
+ repost

Incomplete:

- users
- files
- interactions
- stream marker
- tokens (app/user)
- mutes
- blocks
- channel//message

And token-only read endpoints:

In Progress:

+ user//mentions
+ post//replies
+ post/stream
+ users/me/interactions
+ token

Incomplete:

- user//posts
- user//following
- user//followers
- user//muted
- user//blocked
- user//reposts
- user//stars
- user/me/channels
- users/me/channels/pm/num_unread
- channel//subscribe
- channel//mute
- channels/messages
- users/me/messages
- files/
- users/me/files
- files//content

##Phase #3 - Streaming
I feel that not many important App.net clients embrace user or app streaming (outside push notifications). So we can delay this peice until phase #3.

- user streaming
- app streaming

##Phase #4 - API Compatiblitilty Completion
- Search / Place
- Explore stream
- local oEmbed processor
- WebIntents
- WebFinger
- RSS Feeds
- Password flow authentication
- identity delegation

##Phase #5 Beyond App.net
- what did you want to see in the API?
<https://github.com/neuroscr/AppDotNetAPI/wiki/Future-API-ideas>


Config Roadmap
======

######Standalone mode:
Disables any App.net connectivity. Serves its only social network to any
App.net API supported client
1. Participate in the App.net social network
2. Standalone social network

######port settings
web server port

######api base root url
Where does this server live.

######Upstream client id & secret
if connecting upstream to App.net, you'll need to set this to allow user authorization and app streaming with the upstream network.
######Upstream user token
if receiving a post from spoke API and we need to send it to a parent network, we will have to force all these posts to be under a single user's token. Not ideal but better than nothing

######Rate limtis
Global, verified user, user. Read/Write.

######Anotation Storage limits
/config stuff

######MySQL & Redis connection information
What's enabled? Host, user, pass, etc

######Factual API key

######Allowed client IDs
Limit 3rd party client connectivity

######Additional Protocol Modules:
these would allow additional pushes of streams.

1. tent.io API
2. pond API
3. maidsafe.net API
4. Activity Streams
5. PuSH/RSS
6. Webfinger
7. Diaspora
8. pump.io
9. Ident.ca / StatusNet
10. App.net extensions (what new API features can we add?)
11. Twitter

Potential Issues
======
- Authentication -
Each user in each spoke needs to authenticate with the parent network. For a more p2p system, we really should look at tent.io, pond, or maidsafe.net to see how they handle auth. I'm sure UUIDs are used somewhere.

- Client Secrets & IDs -
To make a client compatible to the real App.net and our API, they will not likely change their client ID. This is not a big deal since apps like patter or vidcast have to expose these in their source that's publicly viewable but we should take care to make sure we don't expose anything unnecessarily.

- Location services - 
While factual has an API, each instances would need their own license.
This feature we may not be able to easily support.
