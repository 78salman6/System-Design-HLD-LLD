# Video Conferencing App High Level Design

I am listing down the functional and non functional requirements which I think is important. Feel free to extend this model as per your understanding and don't forget to give feedback to me - **78salmankashif6@gmail.com**

### Functional Requrements

- User can do 1 to 1 call
- User can start group call / meeting
- User can share Audio/Video/Screen Share
- User can start the recording
- View Placed orders
  > Note - **Video sharing or Screen Sharing** are fundamentally same. Stream of video content having different source. **Video** - Camera; **Screen Sharing** - Laptop screen

### Non Functional Requirements

- Super fast
- High Availability
- Data loss is ok

#### Convention

- All components in green are user interface
- Black color is Load Balancer ( authentication, authorization etc as well )
- Blue components are core components
- Component in Red are mostly related to data like databases, cache, messaging queue, big data etc ( in actual HLD )

### HTTP/HTTPs and Web Socket based

![plot](./diagrams/http_websocket.png)

**TCP** - http/https are TCP based. client first establishes connection with the server ( **TLS Handshake** ). TCP is a **lossless protocol** It retries to send lost data if client did not receive **acknowledgement** ( I encourage you to learn how TCP works if you do not know the internals. ). This retry of TCP slows down the communication. As in the non functional requirement we mentioned that it is okay to have some data loss until and unless it is realtime.
Why TCP makes it slow

1. TLS handshake ( overburden for the communication )
2. TLS retry mechanism makes communication slow ( If it does not receive acknowledgement )
3. Header size of TLS is also bigger than UDP ( around 20 Bytes ) and it makes a bigger chunk.
4. TLS congestion control mechanism -> When client sees a congestion in the network it slows down the rate of sending data.

We can leverage **UDP** protocal for video calls because we are okay with data loss. Why **UDP** is fast

1. Client keeps on sending packets irrespective of the fact whether packet is reaching server ( This is okay because we don't want data consistency, we want video to be real time )
2. Header size of UDP is 8 Byte ( < 20 Bytes )
3. No congestion mechanism
4. No handshake

> Note We will use **TCP** for all other communication b/w client and server which does not include **video transfer**.

#### Explanation on how video call starts

![plot](./diagrams/connector.png)
