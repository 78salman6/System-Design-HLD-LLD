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

10. TLS handshake ( overburden for the communication )
11. TLS retry mechanism makes communication slow ( If it does not receive acknowledgement )
12. Header size of TLS is also bigger than UDP ( around 20 Bytes ) and it makes a bigger chunk.
13. TLS congestion control mechanism -> When client sees a congestion in the network it slows down the rate of sending data.

We can leverage **UDP** protocal for video calls because we are okay with data loss. Why **UDP** is fast

1. Client keeps on sending packets irrespective of the fact whether packet is reaching server ( This is okay because we don't want data consistency, we want video to be real time )
2. Header size of UDP is 8 Byte ( < 20 Bytes )
3. No congestion mechanism
4. No handshake

> Note We will use **TCP** for all other communication b/w client and server which does not include **video transfer**.

#### Explanation on how video call starts

![plot](./diagrams/connector.png)
For sending request for video call **Web Socket handler** is being used ( please bear with me I will explain the details in the coming section ). After receiver receives video call communication moves to **UDP** approach.
**Connector** - Connectoer helps user's machine to find to their public IP address. As drawn in the above diagram. **U1** sends request to router to get its public IP address, router / NAT uses it's public IP address to send this request to **Connector** ( If you do not know details of how NAT works please do google and read. It's pretty interesting no joke on this part ). Connector knows this request is coming from router with IP address **A.B.C.D** and from **port P1** it will tell NAT that this request is coming from P1. So this way U1 will know it's publicly identified **IP address** is **A.B.C.D:P1**. Similarly U2 will also identify its public IP address. Once the public IP address is identified they will share with each other using **Web Socket** based approach.

##### Group calling

All groups <= 5 users ( Small group )
All groups > 5 users ( Large group )

**Small Group** Each user has to send message to each of the member in the call separately. Bandwidth will be shared.
For N users in the group call, each user's bandwidth utilization will increase by ( N-1 ) times.

**Large Group**
![plot](./diagrams/call_server.png)
User will only send video stream packets to **Call Server** then call Server will be responsible for sending to all other users in that call. Call Server's bandwidth utilization will increase and User's bandwidth will only be utilised once inplace of in previous case it had to send packats to each user ( N-1 times). So with slow bandwidth as well user will be able to communicate seamlessly. We can increase bandwidth of Call Server that is in our control.
We have to route it through call server if we want to record on going call. In Peer to Peer communication there is no way we can record the call. ( Practical example - I use Microsoft teams on day to day basis. If you have used it then you might have observed that when you call your colleague then you do not see **Start Recording** option ( Peer to Peer ) and if you join meeting then you see **Start Recording** option ( Through call Server ). Intersting right!! ).

## High level diagram

![plot](./diagrams/video_conferencing_hld.png)
