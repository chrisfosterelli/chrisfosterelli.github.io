How SMS Works

We use it every day, and some of us use it *alot*, but very few of us know exactly how it works. Even those of us that are technologically skilled can sometimes have no idea how the phone they hold in their hand works. Modern smartphones are miniture computers. They have a processor, RAM, and graphics card. That's simple. However, what about the more foreign technologies? In this article I will give an overview of the SMS system and it's siblings. This article will mainly cover a brief overview of technical concepts, without loading you down on implementation details.

<!-- Content Breaker -->

## The SMS Message

Much like the web, SMS is a system that has evolved over time. Because of that, the best way to start with SMS is to start at the beginning. SMS, or "Short Message Service", was initially proposed in 1982 along with the concept of the [GSM network]. The idea came to fruition a couple of years later. The GSM system was designed with a full focus on telephony. SMS is designed to piggyback onto the already existing hardware by transporting messages on the signalling paths used to control telephone traffic. SMS transactions occur when the signalling paths are not being used for managing calls, which allows these SMS messages to have a very minimal cost for providers. All that was required of telephony networks to support SMS was a software update. While sort of a hack, the widespread adoption of SMS that followed was due to its ease of implementation for providers.

Even if you are not using your phone, it is in constant communication with a cell tower. This is known as the "control channel". The control channel is used to periodically send packets between your phone and the tower so that both sides know everything is OK. This update happens as often as every 8 minutes. When someone dials your number, this is how the cell tower notifies your phone to start a call. This is done through the [MAP Protocol] of the [SS7 standard]. In the MAP protocol, certain headers are set to specify that we are sending an SMS message. The payload of these messages is a total of 140 bytes, or 160 latin characters.

This is also where many of the inherent limits of SMS come from. Because messages must fit into this signalling path format, SMS messages are limited to 160 characters. This limit is still in place today.

When sending an SMS message, 

## Concatenated SMS

If you have any sort of modern phone, you probably know that you can send text messages beyond 160 characters. This was a development (hack) that allowed phones to take 6 bytes of the 140 byte payload and use it as a "UDH" (User Data Header) field. This field specifies that the message is a concatenated SMS, or CSMS, message and gives information regarding what piece of the message it is. While this reduces the total text message length to 153 characters, it allows stringing them together. Modern smartphones handle this transparently, so it appears to the user that they can write infininte length messages. In theory you may creates messages of 255 * 153 characters with CSMS. In practice, delays in the messaging network and deliveray problems makes this unlikely to work well. Many phones will switch to a MMS message after a message length of 5 or 6 CSMS parts. CSMS is sometimes called [PDU Mode SMS].

## Scaling The System

When SMS started to become popular, stuff happened.

## MMS and EMS

## SMS options
Read reports.
Broadcasting.
Silent SMS.

## The Cost of SMS

$$ to send a video over SMS
