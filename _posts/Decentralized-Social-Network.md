---
layout: post
title: "Decentralized Social Network"
author: "Krithik Vaidya"
tags: [ipfs, decentralized, nodejs]
image: /Decentralized-Social-Network/cover.png
---


## Introduction

This article talks about how we built a Decentralized Social Network. The project spanned over four months, and was worked on by three of us at the IEEE-NITK Student Chapter. My interest in the project was primarily from a technical standpoint - i.e., how we'd go about solving the different technical challenges that come up in such a system which doesn't a central point of communication. This article talks about the traditional centralized model of the web, what decentralization is, the IPFS library, and the architecture of our application. The link to the source code can be found at the end of the article. Some inspiration for this project was drawn from this [paper](https://courses.csail.mit.edu/6.857/2019/project/17-Foss-Pfeiffer-Woldu-Williams.pdf).

## Some drawbacks of the centralized model

<figure class="image" style="text-align: center; color: gray;"><img src="https://miro.medium.com/max/313/1*ELrZ1WMi7TvGNY_bfa2nLQ.png" alt="Centralized Model"><figcaption>A Centralized System. <a href="https://medium.com/@DeepLearningBitcoin/permissioned-and-permissionless-blockchains-bd710ee686cf">Credits</a></figcaption></figure>

The traditional, centralized model of the web (which is widely in use today and has been in use for many years now), we have websites that are identified by unique links and centrally hosted. A request is sent from our client wishing to see the contents of the website to the remote server, and the server serves the requested content to us.

There are a few important drawbacks of this model of the web. Notably, due to centralization, if the host server faces any kind of system or network failure, the website would be impossible to access for anyone, anywhere. There are a wide variety of reasons due to which a centralized server may go down, such as being attacked by hackers, hardware/software issues, negligence, calamities of nature, etc. 

Another issue with this model is the presence of a central authority of control. This authority has complete control over the data stored on the server, can make changes to the data as they please, and generally have all kinds of exclusive access to this data. This may pose data privacy issues.


## What is Decentralization

In Decentralized systems, there is no single central location of data. Instead, multiple copies of the data are present on different nodes on the network. This solves many of the issues that are faced by a centralized model, by removing a single point of authority, ensuring total control and privacy of data, while making sure that there is no single point of failure. However, this comes at the cost of many implementation challenges, which are dealt with in pretty interesting ways. 

## IPFS

Our project is built atop the [IPFS](https://ipfs.io/) (InterPlanetary File System) framework, which is a peer to peer filesystem and stack of network protocols enabling file access and distribution on the decentralized web. The files here can be anything – text files, images, video, entire folders containing any types of files, etc. Till date, over 5 trillion files have been hosted on the IPFS. Every system on the IPFS network is called a 'node' or a 'peer', and each node has a unique identification string associated with it.

The implementation of IPFS itself is a very vast topic. In a sentence, IPFS can be said to be the evolution and fusion of technologies like peer to peer protocols such as BitTorrent, and distributed, versioned file systems such as Git. Using these technologies and improvements upon these technologies, it provides us state-of-the-art decentralized hypermedia sharing.

#### How does IPFS locate and retrieve content without a centralized server?

IPFS allows each user (peer) to host whatever data they'd like locally. Every file stored on the IPFS has a unique hash value (that is generated through cryptographic signing), called as the **Content Identifier (CID)** (a long string, like for eg., QmwArsqEwjUoLOq2gVtTUg2tC3jNf5Htm2ZeO4rKnsFF3FDp46A). The CID is calculated from the contents of the file (or folder), and is a unique string for a given file. Two files with the same content but hosted on different nodes will have the exact same CID. 

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Decentralized-Social-Network/3.png" alt="CID sample"><figcaption>CID for a sample file, <em>im.png</em></figcaption></figure>

This forms the basis for content-based addressing that is used by IPFS to locate data on a decentralized network. Since files are identified by their content and not their location (for e.g. with a URL), this system is called “content addressing”. A node on the IPFS network can ask all the peers it’s connected to whether they have a file with the required content identifier, and if any of them does, they respond with the file. This wouldn't be possible without a quick to compute, short and unique cryptographic hash for every file.

An important provision that IPFS ensures is that multiple peers host the same files, which ensures that - 
- If any node fails, all its data is not lost; there will always exist other nodes that have copies of its data.
- Suppose we wish to transfer files from a node that is geographically located at a very far distance. Accessing the server may take a while at first. However, once all the data is downloaded on the client node, all other nodes that are close to it but geographically far from the original source node can get the content from this node itself, thus increasing response times.

An important point is that any small change in the data of a file will result in a CID that is completely different from that of the original file, ensuring that the user gets exactly the file they requested for, and can verify its authenticity (data integrity).

## A deeper technical dive

To understand how exactly IPFS internally stores content, identifies different nodes on the network, etc., please check out the [IPFS documentation](https://docs.ipfs.io/concepts/).

## IPFS benefits

These are some of the benefits of using IPFS:
- No censorship of data access
- Data authentication and encryption
- Lower bandwidth and costs
- Reduced latencies when a large number of nodes are participating on the network
- Provides a free, incentivized way to distribute our content, hence ensuring its longevity
- Features for working offline and automatically syncing changes once online

## Our social networking application

Now, let's actually come to our project. We aim to develop a decentralized, social network application over IPFS that supports all the important functionalities that a standard social network today does. For the frontend of the application, we use HTML, CSS, Bootstrap and Javascript to ensure a intuitive and responsive web design. For interfacing with the IPFS library, we use the NPM ipfs package. To access NPM packages in the client-side (since there is no central server, and every client is their own server), we use Browserify. The project also uses Orbit-DB, a decentralized database that uses CRDTs (Conflict-free Replication Data Types) for storing and syncing records of users of the social network across nodes.
To uniquely identify indivudual nodes on the network, we use the IPFS peer id that is associated with each node.

When a node joins the social network, a root_folder is created in the IPFS filesystem of each node. This root folder contains all their files related to the social network such as the friends list, all created public/private/friend posts, folders for each friend, etc.

For the functionality of adding friends, 

When a user joins the decentralised social network, the following steps happen on their local system as shown below:

![Startup](/blog/assets/img/Decentralized-Social-Network/1.png)

- On starting up our decentralised social network web-app, the user starts up their node on the IPFS network
- On starting up their node in the IPFS network, a unique peer ID is generated which is used to identify each user (once this is generated for a particular user, it never changes).
- After the peer id is generated all the necessary information is loaded. Every node has a root_folder on their IPFS filesystem for storing all their files related to the social network. This root folder contains the friends list, all created public/private/friend posts, folders for each friends, etc.
- Then IPFS tries to connect to all the peers in the node's friends list.
- Then the node connects to the Orbit chat network to listen for messages and enable messaging services

The various features of the decentralised social network are shown below. The features can be classified into 3 broad categories - adding friends through usernames, viewing posts and chatting. 

![Features](/blog/assets/img/Decentralized-Social-Network/2.png)

The implemented system has no central location of data and instead multiple copies of data are present on different nodes of the network. Accessing a person's data first involves mapping their username to the root of the person's file system hash and accessing their data from that. The mapping pairs of username - root folder CID is stored in a decentralized database, Orbit-DB. When a node joins a network, it immediately adds the username - root folder hash pair to the database.

Whenever there is a change in the root folder's contents (i.e. when the user adds/modifies/removes a post or adds/removes a friend), the root folder hash gets entirely changed. This means that the current hash stored in the Orbit-DB is outdated. Hence, the node needs to simultaneously update its record in the database whenever it makes a change in the root folder. This can lead to non-trivial issues in consistency of the data in the decentralized database.

For the functionality of friending, the initiating node (node1) should create a folder which is named with the username of the friend node (node2) in it's IPFS filesystem. It will also add node2 to it's friends list file. node2 will then get the root folder hash of node1 by mapping node1's username to root folder hash using the Orbit-DB database. When node2 sees a folder with it's own username in node1's root folder, it understands that node1 wants to be it's friend. It adds node1 to its friends list and creates a folder for node1 in its own root folder. This is the process of friending.

When a node starts up the decentralized social network, IPFS will internally try to create a P2P connection to all nodes in the friends list.

For the posts functionality, they can be classified into 3 types - private, public and friends post. 

- Private posts can be sent from 1 user to another exclusively without any external interference. A node1 that wishes to send a private post to node2 will create a new file in node2's folder present in its (node1's) root folder. This folder would have been created during the friending process. Then, node2 can access this folder and read the post.
- Public posts can be viewed by anyone who searches the username of the desired user. Every node has a folder in which it places the files for public posts in.
- Friends posts are broadcasted to all the users in the friends list of a user.

Normally, all the above posts except public posts would be cryptographically secured to prevent non intended nodes from reading the posts.

The base paper suggested the use of PubSub (Publish/Subscribe) for the messaging system. After testing and considering the not-very-reliable nature of the system, we decided to omit the usage of PubSub and instead built our chat functionality upon an exising chat library (Orbit-Core). Additionally, we have added username and group chat support.


## Conclusion

In this article, we have explored the basics of Decentralized apps and the IPFS framework. An overview of the architecture of our implementation of a Social Network in a Decentralized setting was also given.
The repository for this project can be found [here](https://github.com/IEEE-NITK/decentralized-social-network/), which also contains screenshots of the application in action. The source code is heavily commented for ease of understanding.

## References

[https://medium.com/zkcapital/ipfs-the-distributed-web-e21a5496d32d](https://medium.com/zkcapital/ipfs-the-distributed-web-e21a5496d32d)  
[https://ipfs.io/](https://ipfs.io/)  
[https://github.com/ipfs/js-ipfs](https://github.com/ipfs/js-ipfs)  
[https://orbitdb.org/](https://orbitdb.org/)  
[https://courses.csail.mit.edu/6.857/2019/project/17-Foss-Pfeiffer-Woldu-Williams.pdf](https://courses.csail.mit.edu/6.857/2019/project/17-Foss-Pfeiffer-Woldu-Williams.pdf)  