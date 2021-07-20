# Summary

Adding native support for the Secure Reliable Transport ([SRT](https://github.com/Haivision/srt)) protocol to OBS Studio instead of using the ffmpeg implementation of SRT can unlock additional features and make it easier to use.

# Motivation

The open source SRT protocol has matured over the years and became a de-facto standard for live streaming. Haivision and SK Telecom have started an initiative for standardisation at IETF. More than 450 companies have joined the SRT Alliance and are supporting SRT today. 

The SRT protocol enables low-latency point-to-point and point-to-multipoint streaming. It also plays an increasing role for ingest into CDNs. Recently AWS Elemental MediaConnect enabled SRT ingest, and we are sure that many CDNs and streaming-platforms will follow.

## Highlights of the SRT Protocol

- SRT is content agnostic and open to the use of a wide variety of audio and video codecs.
- SRT offers a very good trade-off between low latency and robustness at high packet loss rates. It is much faster than RTMP and can cope with unpredictable connections via public internet, such as between continents. (See whitepaper [RTMP vs. SRT](https://www.haivision.com/resources/white-paper/srt-versus-rtmp/)).
- SRT can work like RTMP on a single port and handle multiple incoming or outgoing connections, each with an individual stream ID, analog to RTMP's stream key. (See [SRT access control](https://github.com/Haivision/srt/blob/master/docs/features/access-control.md)).
- SRT is supported by many server systems and CDNs already, including Avid, AWS Elemental MediaConnect, Wowza, Nimble Server, AliCloud, Haivision Hub, and many more.
- SRT supports redundant connection modes, enabling even higher reliability over multiple network connections. Active-Active and Active-Passive modes are supported with no need for additional hardware or service providers. (See the [connection bonding documentation](https://github.com/Haivision/srt/blob/master/docs/features/bonding-intro.md) for further information).
- SRT can work with bi-directional streaming, allowing workflows like talkback or bi-directional audio/video communication.
- SRT supports 128/256-bit AES encryption.
- SRT is currently in a standardisation process with the IETF. (See the [RFC submitted](https://datatracker.ietf.org/doc/html/draft-sharabayko-srt-00)
- SRT enables easy firewall traversal with Caller, Listener, and Rendezvous modes.

## Benefits of a Native SRT Implementation in OBS Studio

Currently OBS Studio supports SRT through the underlying ffmpeg implementation. However, this approach has drawbacks in terms of both how the protocol can be used and which of its features are supported.

- All SRT options have to be specified in a command line to be passed to ffmpeg. This is complicated for inexperienced users, and beginners might be overwhelmed by the number of available [SRT API socket options](https://github.com/Haivision/srt/blob/master/docs/API/API-socket-options.md#list-of-options). A UI integration would improve usability and adoption.
- Automatic Reconnect: If an SRT connection is interrupted for too long (longer than specified in "[peer idle timeout](https://github.com/Haivision/srt/blob/master/docs/API/API-socket-options.md#SRTO_PEERIDLETIMEO)"), the socket is closed. The application will be notified and can re-establish the connection. This feature is currently not implemented in ffmpeg and, therefore, not available in OBS Studio. In order to re-establish the connection, OBS Studio has to be re-started completely. Stopping and re-starting a stream through the OBS Studio UI doesn't work currently. 
- SRT statistics: One of the biggest benefits for streaming operators is the [rich live statistics](https://github.com/Haivision/srt/blob/master/docs/API/statistics.md) which SRT provides. Having visibility into a stream's  bandwidth usage, RTT values, packet loss rate, buffer levels and re-transmission attempts makes it easier to troubleshoot problems. SRT can produce graphs to display data issues, making them more easily identifiable. These data can be used to prove stability or other issues with clients or ISPs. This data can also be used in applications to control the actual video encoding bitrate according to network performance, like Haivision does with [Network Adaptive Encoding](https://www.haivision.com/resources/streaming-video-definitions/network-adaptive-encoding/). 
- SRT connection bonding: It is sometimes necessary to explicitly specify the NICs being used for streaming. Instead of looking up NIC names and adding them to the command line for ffmpeg, having a drop-down menu in the UI auto populated with available NICs would be a great help to users.

# Drawbacks

Integration of the SRT protocol natively within OBS Studio would require significant effort.

# Additional Information

If there is any interest, we would be happy to organise a webinar for OBS Studio developers to explain SRT features in more detail and to answer questions.

The SRT community can be contacted through [GitHub](https://github.com/Haivision/srt) or the SRT [slack channel](https://slackin-srtalliance.azurewebsites.net/).

