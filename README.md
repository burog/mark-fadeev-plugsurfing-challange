# BE Coding challenge - Music Service Design document

# Introduction
Music Service is a REST API for providing clients with information about a specific music artist. The information is collected from 4
different sources: MusicBrainz, Wikidata, Wikipedia and Cover Art Archive

# Stakeholders
usually here should be mentioned all stakeholders, and all 3rd parties are impacted by current change.

# Requirements

## Functional requirements
i as a client want to receive the following information about artist by mbid (MusicBrainz Identifier):
- name of artist
- gender
- country
- disambiguation
- description (based on `Wikipedia`)
- list of albums with front album cover art (with links to `Cover Art Archive`)

using following API: `/musify/music-artist/details/f27ec8db-af05-4f36-916e-3d57f91ecf5e`
and expecting to receive following json response:
```json
{
  "mbid": "f27ec8db-af05-4f36-916e-3d57f91ecf5e",
  "name": "Michael Jackson",
  "gender": "Male",
  "country": "US",
  "disambiguation": "“King of Pop”",
  "description": "<p><b>Michael Joseph Jackson</b> (August 29, 1958 – June 25, 2009) was an American singer, son
  "albums": [
    {
      "id": "500d9b05-68c3-3535-86e3-cf685869efc0",
      "title": "Farewell My Summer Love",
      "imageUrl": "http://coverartarchive.org/release/8172928a-a6c7-4d7c-83c8-5db2a4575094/13404444760.jpg"
    },
    {
      "id": "37906983-1005-36fb-b8e7-3a04687e6f4f",
      "title": "Anthology",
      "imageUrl": "http://coverartarchive.org/release/a7a74484-8c25-47e3-9afc-7de701ad3dde/1619836290.jpg"
    },
    {
      "id": "1516d16c-fad9-413f-99ba-ad1f767c608a",
      "title": "The Motown Years...His Greatest Hits",
      "imageUrl": "http://coverartarchive.org/release/1abad59d-5f52-4b16-9b48-e8beeaf76ec8/8554032338.jpg"
    }
  ]
}
```

Sequence diagram:
![sequence_diagram.jpg](diagrams%2Fsequence_diagram.jpg)

Component diagram:
![component_diagram.jpg](diagrams%2Fcomponent_diagram.jpg)

### Analytics
it can be done by preparing dashboard on top of data pulled on traces by OpenTelemetry

### Out of scope
- user management

### Constraints and assumptions:
- assume for images front cover will be enough
- do not store binary data of album covers
- service is going to provide synchronous API

## Non functional requirements:
- 10000 requests per second at peak time

I assume peak time will be related to some artist event or release or festival, so it means requests should be quite similar and for that we can use cache in load balancer.

We should use NIO or green (virtual) thread for incoming request, due to resource efficiency of handling a peak time load.

Request limiter can be implemented on Nginx side by limiting number request per ip, session, and specifying timeouts. it is good to have feature to avoid abuse of system

Nginx as load balancer. 
due to :
- High performant web server even as standalone
- it can be configured as HA cluster with Active-Active or Active-passive type

With configured cache policy. Due to the fact that new release for artist it is quite a rare event (1-2 per year), policy can be configured with quite a long lifetime.

# Architecture Overview

# Data Model
All necessary data will pass through message broker. 
The Model should be the same as in result.
The Format is Json, same as expected in a result.

# Security
Due to the absence of user management, no more security is left to be considered. 
We Need to make sure that VPN and firewall are configured properly so only one endpoint is available from the internet (no message broker, open-telemetry).
Https isn't necessary here as it is publicly available information for that specific, and we can save some RPS.

# Scalability
Due to design and absence of state, the main "worker" should be easily scalable. 
Auto-scalable mechanism should be configured. 
Criteria for scalability can be number of incoming requests. 

# Capacity

## Traffic estimates
Lets assume average request is about 2 KB, so 10000 request will be around 20 MB ~ 20 MB/s

## Storage estimates
As it no DB, we need to calculate message broker and persistence for cache.

## Memory estimates
As this solution relies on cache, we need to estimate how much it can require of RAM.
Let's assume that cache should be evicted every week, to keep actual data of new releases.
Let's assume 20 % of all requests will be unique, and for a non-peak period we're expecting 1000 requests. So 200 unique requests per second. 
So for average week it will be 7 * 24 * 60 * 60 * 200 = 120 960 000 request ~ 240 GB total memory  

# Deployment

> **Note**  
> according to the Stack in challenge company except to use AWS lambda, unfortunately i don't have experience with that

- containerize application with docker and AWS image
- rely on AWS and Github solutions for CI \ CD 

# Monitoring and Logging
The Core component of monitoring and logging is open telemetry.
For our service, we need to add javaagent. Detail can be found [here](https://opentelemetry.io/docs/instrumentation/java/).

And to collect all the data we need to it, allow receiving, process and export telemetry data.

# Error Handling
Part of error handling — we need to use circuit breaker and retry mechanisms to keep balance between overloading 3rd party and instability of network.

# Cost estimation
TBD - describe how much the whole solution would cost.

# User Interface (UI) Design
No user interface is going to be implemented. Only API.

# Testing
Unit tests cover individual services (WikiDataService, BrainzService, CoverService, WikiPediaService), while integration tests verify the interaction between services. Automated UI tests ensure a consistent user experience.