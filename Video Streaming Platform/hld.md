# Video Streaming Application High Level Design

I am listing down the functional and non functional requirements which I think is important. Feel free to extend this model as per your understanding and don't forget to give feedback to me - **78salmankashif6@gmail.com**

### Functional Requrements

- Production house can upload video
- User's Homepage + Search
- User can Play Videos ( different quality, extension, dimention )
- Support all types of devices

### Non Functional Requirements

- No buffering ( Low Latency, High Availability )
- Good Recommendation
- User's Session time ( Low Latency, High Availability )

> Note: Suppose there are N supported formats, M types of different dimention screen videos, ) types of bandwidth ( video quality ). Then in total N*M*O videos will be needed.

#### Convention

- All components in green are user interface
- Black color is Load Balancer ( authentication, authorization etc as well )
- Blue components are core components
- Component in Red are mostly related to data like databases, cache, messaging queue, big data etc ( in actual HLD )

### Video Upload HLD from Production house

![plot](./diagrams/production_house.png)

> Note: Client requests chunks of videos while user is watching it requests for other set of chunks. If client realises subsequesnt chunks are not coming well in time, it starts requesting for low quality chunks ( **Adaptive bitrate streaming** )

#### Description of components of video upload flow

![plot](./diagrams/workflow.png)

1. First of all production house will upload video in their machine ( VM or anything ), then in someway they will share the machine details to the platform. Then platform using **SFTP** dumps movie to its own **BLOB storage**. Admin of video also provies **metadata** information ( uploader details, description, lot of images, large variety of tags etc ). These metata information is being stored in **Cassandra**.
2. After successful upload happens in the blob storage, Asset **onboarding service**
   posts an event into **Kafka** about completion of storage.
3. One of the consumer of above kafka topic is **Content Processor** ( **Workflow engine** ).
4. In the **Workflow engine** first of all **File Chunker** will chunk the file at the very beginning. It will break entire video into small size stream chunks ( Refer the diagram for the flow ).
5. For each chunk **Content filter** is called. It filters the chunks based on the criteria ( Violence, Vulgarity, Nudity, Piracy etc ). All filtering can be done parallely ( using many instance of Content filter. Different intance may parallely work on differnt different chunks ). After the processing is done it posts the event into Kafka.
6. **Content Tagger** listens to above Kafka topic and does the tagging parallely on different different chunks. It creates relevant tags and **thumbnails**. ( We will discuss it in detail in near future ). After compeletion of the process it posts event as Kafka topic.
7. **Transcoding** listens to above published topic. It converts chunks into different differnt format. Ex - AVI, MP4 etc. After compeletion of the process it posts event as Kafka topic.
8. **Quality Converter** listens to above published topic. It converts chunk into different different quality stream. Ex - 144 p, 244 p, 480 p, 720 p etc. After compeletion of the process it posts event as Kafka topic.
9. After the above processing, workflow engine starts to accumulate chunks of same video and it starts to upload it into **Content delivery network** ( CDN ). Then after successful completion of video upload into CDN, it posts event into Kafka. **Notification service** uses this event to send notification to production house about successful upload of their video. **Serarch consumer** will also listens to this completion event and makes this video searchable by the Users.

### User facing product

![plot](./diagrams/user_side.png)

#### Description of components of User side

**User Service** - Source of truth of all user related information. Lot of times many other services are calling user service into **Eco System** for the details of a particular user. Every time calling **MYSQL** will slow down the performance. So to reduce latency we can use some caching ( Reids, LRU cache ). Cache will hold all the details of user ( payload ) with key as **user_id**. **Serach Service** can call user service to get age of user to filter content based on age.

If analytics says people are mostly going on next page and not viewing video of 1st page ( depicts bad recommendation ). It gives area of improvement ( Both for home service as well as Search Service ).

> Note: All information about CDN's, videos lie in **Cassandra**.
> **Host Identity Service** - It knows which video you want to play and they know from where you are coming.
> Host service queries **Asset Service** to get all **CDN's** that are available with this video which user wants to view and given the location of the user and the location of all the videos that are there try to come up with one of the CDN that is responsible to play this particular video. They will assign CND which is clouse to geographical location of user.

From analytics if eco system comes to know people of India is watching **Money Heist** very much. We could actually host a CDN in India and upload that video to that CDN to reduce the latency.

**Stream Stats Logger Service** - If video is closed after 10-20 seconds then it is probably not a good video. But if lot of people are watching more than 90% of video content it can be classified as good video

### Home Page and Search HLD

![plot](./diagrams/Home_Search.png)

> Note: ELK provides **fuzzy searching**. ELK query User Service to get info about user to do some age level filtering.
> **Tagger Service** - Puts all different tags in Kafka topic. Spark streaming service does processing then it selects let's say 100 tags and post it in Kafka topic. **Asset Service** stores those tags as metadata of movie in **Cassandra**.

#### Discussion on why Cassandra

1. This is the database which can handle massive reads and writes.
2. It follows **no master** kind of strategy ( Ring of lot of nodes ). Each node stores some part of data ( each node knows what particular data resides in which node ). Cassandra is very good if you have small number of queries ( on some partition key ). It is bad at **Random queries**.

> Note: Tageer service also creates many **thumbnails** ( Stored in **Hadoop cluster** ).

#### How to choose best thumbnails

When search results are shown to user, initially random thumbnails are shown.
Suppose a particular thumbnail ( let's say containing Action poster ) is mostly hit by group of people with age range between 18-25. Then Recommendation service will collect this thumbnail as one of the best thumnail for the group of people in the range 18-25. For same movie let's say thumbnail with Family poster is clicked by most of people in range 35-45 then show this thumbnail to that user category.

#### Form cluster of users ( Using ML model)

Liking certain kind of videos makes cluster ( K mean or any other cluster forming algorithms ).
**Cluster based on Genre**

1. For each genre use other people in the cluster to find out what are the movies they watched / liked. Use that to build genre based recommendation.
2. Searches user are doing - Whether that movie are in data store or not. Show them similar kind of movies ( in search result as well as in the user's home page ). If search for a movie crosses certain threashold try to onboard that movie into the platform.

> Note: If based on analytics system can predict all the list of items customers can watch tomorrow, we can actually load the list in **Local CDN**. It will save a lot of bandwidth and will reduce the latency drastically.

> Note: If new TV series is launching in **Hindi**. People of country where Hindi tounge people lives are the target audience. Target those regions.

**CDN Writer** - Suppose CDN writer came to know Local CDN needs following video
C1, C2, C3, C4, C5, C6
Current content in Local CDN -> C1, C2, C3, C8, C9, C10
It will delete C8, C9, C10 and will write C4, C5, C6 into the Local CDN.

#### A Discussion on Decentralization

Suppose videos -> C1 - C25 needs to be uploaded in many Local CDNs ( LC1, LC2, LC3, LC4, LC5 ). It will be uploaded from Main CDN ( It can be Amzon S3 / Microsoft Azure BLOB ).
1st way is to push all 25 videos from Main CDN to all the Local CDN. In this case BLOB storage will be in high stress and it may cause **Single Point of Failure**.

A better way is push C1 - C5 into LC1; C6 - C10 into LC2; C11-C15 to LC3; C16 - C20 to LC4; C21 - C25 to LC5
Then let local CDNs to syncup ( something like how torrent works ) among themselves. This push and syncup operation may happen at the time of low traffic. Like from 3 AM in the morning.

**Advantage of Decentralization**

1. No Single point of failure
2. Low cost
3. Transfer during low traffic
