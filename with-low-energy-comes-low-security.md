# Bluetooth: With Low Energy Comes Low Security

* [Paper and Presentation](https://www.usenix.org/conference/woot13/workshop-program/presentation/ryan)

## Introduction

* "Given that the target devices for BTLE are expected to have low computation
  capabilities, compromises were made to simplify the protocol."
	* Dependent on timelines, this is likely a huge understatement. If 
	  I understand correctly, BLE just nixed the idea of a key exchange
	  in favor of basically sending a key in cleartext.
* This paper intends to cover:
	* BLE sniffing
	* BLE packet injection
	* BLE crypto flaws (key recovery!)
	* The hardware/software platform that can be used for BLE attacks and monitoring.

## Bluetooth Low Energy

* BLE PHY and link layers are incompatible with BT Core.
* The PHY layer of BTLE uses Gaussian Frequency Shift Keying.
	* TOOD(kkl): Read about GFSK.
* BTLE has 40 channels. 37 data channels. 3 advertising channels.
* Advertising channels are used for broadcasting and connection establishing.
* Each packet begins with an 8 bit preamble and a 32 bit _access address_ (AA).
	* AA uniquely defines a connection.
	* Adversing channel has an AA of 0x8e89bed6
* Following the AA is a variable length _Protocol Data Unit_ (PDU) that contains
  the message payload.
* Finally, all packets end with a 24 bit CRC.
* We have the following packet structure.

	| 8 bits   | 32 bits             | variable length (2 -> 39 bytes) | 24 bits | 
	------------------------------------------------------------------------------
	| Preamble | Access Address (AA) | Protocol Data Unit (PDU)        | CRC     |

* Other BT Core aspects are simplified in BTLE

## Eavesdropping

* The ability to passively sniff BTLE communications had existed prior to the paper.
* It seems these solutions required sniffing _at connection setup_. The authors were more clever and came up with ways to derive the setup parameters for sniffing even after a connection was already established.
* The parameters that are needed (some are optional but nice) to sniff BTLE communications but are public after connection establishment are as follows:
	1. Hop interval (aka dwell time)
	2. Hop increment
	3. Access address (AA)
	4. CRC init

### Ubertooth
* They describe the hardware and its capabilities a little bit.

### From RF to Bytes
* The Ubertooth does not look for a preamble to identify backets, it insteads looks for the Access Address.
* This is easy for advertising (its fixed)
* The AA is otherwise only transmitted during setup. The authors allude to a way around this.

### Following Connections
* Due to hardware limitations, the Ubertooth must jump channels in lock-step with the target device.
* BTLE does channel hopping for each packet it sends.
* Channel hopping is done with the following pattern:
	
	`nextChannel = channel + hopIncrement (mod 37)`

* Master and slaves hop to same channels at the same time. They send empty PDUs (with header and crc and stuff) if they don't want to send data at a given interval.
* The Ubertooth does this channel hopping.

## Promiscuous Mode
* Ubertooth assumes all 37 channels are used (even though its not required). Anecdotally, they have never seen hardware use less than the 37 channels.
* In order to sniff packets the previously defined 4 values should be known.

### Determining the Access Address (AA)
* Empty data packets have a predictable form and are sent frequently.
* Data packets consist of a 16-bit header, 0-37 bytes of payload, and a 24-bit CRC.
* Empty data packets just have 16 bits of header and a 24-bit CRC.
	* Two-bits are varying (SN and NESN bits) and the rest are zero.
* To identify an AA, Ubertooth waits for an empty packet (which is easily identifiable)
* This means, the previous 32 bits from the beginning of the payload was the AA.
* The empty data packet detection has a relatively high FP rate. To counteract this, Ubertooth just keeps a data structure of candidates and once they identify around 5 packets with single AA they determine that AA to be valid.

### Recovering CRCInit
* Recoving CRCInit allows Ubertooth to compute CRCs on data.
* This makes other connection parameter recovery steps easier.
* BTLE uses a LFSR as a CRC over the entire packet. The LFSR is seeded with CRCInit.
* The BTLE LFSR is reversible.
* Once a candidate packet is received, the LFSR is reversed and a CRCInit candidate is determined.
* Just like the AA, to reduce FPs this CRCInit candidate must be found multiple (5ish) times to be treated as legitimate.

### Hop Interval
* The hop interval is derived using timing!
* Since there are 37 channels, the Ubertooth sits on a channel and times
  the gap between packets on that channel (i.e. the channel hops have cycled)
* A simple formula can recover the hop interval based on timing.
* Again, for FPs this hop interval must be noted multiple times.

### Hop Increment
* Hop increment is derived using timing based on packet arrival on two channels.
* In other words, Ubertooth listens on channel X and receives a packet. It starts
  a timer and hops to channel Y. When Ubertooth receives a packet on Y, it can
  do solve for the hop increment using algebra (and a lookup table)

## Injection
* Ubertooth can do BTLE injection.

## Bypassing the Encryption
* The BT SIG invented their own key exchange.
* The BTLE key exchange works as follows:
	1. A temporary key (TK) is generated. TKs are derived using information sent over BTLE in the clear and a pairing secret. The pairing secret depends on the pairing mode (e.g. "Just Works", 6-digit pin, OOB).
	2. A confirm value is calculated. The paper doesn't cover this, but I believe its just a challenge response thing (e.g. Whats the random value I encrypted? You should know since you have the TK.)
	3. A TK is used to encrypt a short-term key (STK) and is sent over BT.
	4. The STK is used to encrypt the long-term key (LTK) and sends that over BT.
* The authors wrote code to break the TK. A passive eavesdropper can do this offline and retroactively decrypt BTLE comms.

### Mitigations and Counter-Mitigations
* BTLE may mitigate some of the "key exchange" sniffing attacks with two observations:
	* A key exchange only happens at the beginning of a connection. After a LTK is established no key exchange is done
	* BTLE connections use a session-specific nonce in addition to the LTK. Only sent at the beginning of a connection.
* Both can be mitigated using two techniques:
	* `LL_REJECT_IND` packets are a BTLE packet that forces renegotiation.
	* Injecting random noise may kill the connection and force renegotiation of a session.
