# DNSSEC

## Task 1: DNS Insecurity

1. In what context was the DNS cache poisoning attack discovered? (Who, when, why)
2. How does this attack work and how to prevent it?
3. What is DNS spoofing and what tools can be used to integrate this attack?
4. What does a validating resolver do? What is the difference between island-of-trust versus full-chain-of-trust?
5. Does DoH substitute DNSSec or complement it? What are the differences between them?
6. What do you think about DoH vs DoT vs DNSCrypt?

--------------

### Answers:

1. &  2. In 2008, researcher Dan Kaminsky understood that there were only 65,536 possible transaction IDs. An attacker could exploit this limitation by flooding a DNS resolver with a malicious IP for a domain with slight variations—for instance, 1.google.com, 2.google.com, and so on—and by including a different transaction ID for each response. Eventually, an attacker would reproduce the correct number, and the malicious IP would get fed to all users who relied on the resolver. The attack was called DNS cache poisoning because it tainted the resolver's store of lookups. By exponentially increasing the amount of entropy required for a response to be accepted this attack was prevented. Additionally, lookups and responses traveled only over port 53, the new system randomized the port-number lookup requests used. For a DNS resolver to accept the IP address, the response also had to include that same port number. Combined with a transaction number, the entropy was measured in the billions, making it mathematically infeasible for attackers to land on the correct combination.

3. DNS spoofing is an attack in which altered DNS records are used to redirect online traffic to a fraudulent website that resembles its intended destination. Methods are : 
    * Man-in-the-middle duping:
        * The interception of communications between users and a DNS server in order to route users to a different/malicious IP address.
    * DNS server compromise:
        * The direct hijacking of a DNS server, which is configured to return a malicious IP address.

DNSSEC can help protect against DNS spoofing.

![](https://i.imgur.com/QEwfRpG.png)
<p align=center>
<i> Figure 1: DNS server compromise attack</i>
</p>


3. After enabling the DNSSEC, a validating recursive name server (a.k.a. a validating resolver) is responsible for checking the integrity and authenticity. So, it will ask for additional resource records in its query, hoping the remote authoritative name servers respond with more than just the answer to the query. If DNSSEC responses are received, the validating resolver performs cryptographic computation to verify the authenticity (the origin of the data) and integrity (that the data was not altered during transit) of the answer and even asks the parent zone as part of the verification. It repeats this process of get-key, validate, ask-parent, and its parent, and its parent, all the way until the validating resolver reaches a key that it trusts. In the ideal, fully deployed world of DNSSEC, all validating resolvers only need to trust one key: the root key.

4.  When we are talking about the Island of trust, it means that the trust anchors are published in a dedicated domain. Whenever a validating resolver recognizes that a zone is signed it will first try to validate it by assessing if it is within the island of trust configured by its local trust anchors. When the validated domain is not in a trusted island the resolver will lookup perform a lookup and use the trust anchor from that zone if and when available. So, instead of the trust anchor being a root name server, it is the uk-DNSSEC domain. This could represent something as restricted as a corporate domain with internal child domains. As other levels in the DNS hierarchy transition to DNSSEC, these islands can easily establish a trust relationship with them by exchanging keys. But, in the full chain of trust, we will use a root public key and try to resolve and authenticate other domains through reverse direction till we get to the root.

5. DNS over HTTPS is a standard developed for encrypting plaintext DNS traffic to prevent malicious parties from interpreting the data. So, it will complement DNSSec. DNSSec will try to keep records from forgery but (signing and checking the authenticity), by using DoH, we will be able to keep safe the DNS packets' content from outside. DNSsec protects resolvers while they getting data from authoritative servers, DoH provides (in addition) privacy for clients while they getting data from resolvers

6. The main difference between DoT and DoH is the layers at which the encryption is enabled.  DNS-over-HTTPS is applied at the application layer, while DNS-over-TLS is applied at the transport layer.  
DoH is not necessarily better than DoT, and in a lot of ways, DoT is more efficient because of which layer within the TCP/IP model it is enabled. Because of this reason, DoH will need more coding, more libraries and packet sizes are larger than DoT.
DNSCrypt is another DNS encryption option and like DoT, it is applied at the transport layer. DNSCrypt prevents localized man-in-the-middle attacks and encrypts DNS traffic between the user and recursive DNS servers, though it does not provide end-to-end encryption. 

---

## Task 2: Validating Resolver

1. Setup
    * Enable DNSSec to your BIND or Unbound and verify the root key is used as a trusted source.
    * Use dig or drill to verify the validity of DNS records for isc.org, sne21.ru

2. Validate
    * How does dig or drill show whether DNSSec full-chain-of-trust validation was successful or not? Hint: it’s about a Flag.

3. Counter-validate (breaking the chain of trust)
    * Where does BIND or Unbound store the DNSSec root key?
    * How do managed keys differ from trusted keys? Which RFC describes the mechanisms for managed keys?
    * Modify the DNSSec root key and test your validating resolver.
    * How did your server react?
-----


## Answers:

1. For enabling DNSSEC, I used the unbound-anchor tool for unbound. By this simple command, a root.key file created in /usr/local/etc/unbound directory. This file is the root anchor file that is used by Dnssec to validate the signing trusted chain. 
```
unbound-anchor
```
After that, I needed to enable the automatic updated anchor statement inside the unbound.conf file to keep the anchor updated if it is changed  

Then, I used dig to verify the validity of DNS records for isc.org.


![](https://i.imgur.com/Makv6MH.png)
<p align=center>
<i> Figure 2: dig isc.org after enabling DNSSec </i>
</p>

2. For validating, we need to look at the returned flags. We have the "AD (Authentic Data)" flag that indicates the resolver believes the responses to be authentic - that is, validated by DNSSEC


3. * Unbound will keep DNSSec root key in /usr/local/etc/unbound/root.key directory. 
    * When we set the auto-updating root keys, the unbound itself is responsible for obtaining the new file and replace it. In this manner, the root.key file is considered as Managed keys. But, we can also download root.key files from where we think are trustable and put it there to be used by unbound for resolving process. In this way, we are using Trusted root.key file. To use this file, we need to omit "auto" from the specific line in unbound.conf (Figure 3). I think RFC 5011  speaks about the mechanism.
    * For this part, I changed the root.key a bit (just by nano and changing some of the contents). And the result was like Figure 4. As it is obvious, after the change, it was not able to build the trust chain so, there is no AD flag in the first dig result.

![](https://i.imgur.com/LScbWrz.png)
<p align=center>
<i>Figure 3: unbound.conf modification to use trused root.key file</i>
</p>

![](https://i.imgur.com/FkLmcyL.png)
<p align=center>
<i>Figure 4: DNSsec results before and after root.key file</i>
</p>

------
## Task 3: Secure Zone

*Island of trust*

1. Look up which cryptographic algorithms are available for use in DNSSec. Which one do you prefer, and why?
2. What is the difference between Key-Signing Key (KSK) and Zone-Signing Key(ZSK)?
3. Why are those separated and why do those use different algorithms, key sizes and lifetimes?
4. Use BIND9 tools (dnssec−keygen & dnssec−signzone) or NSD tools (ldns−keygen & ldns−signzone) to secure your zone. Show the configuration files involved and validate your setup

*Full chain of trust*

5. Does your parent zone offer secure delegation?
6. Describe the DS and DNSKEY records from sne21.ru that are important for your domain. Which keys are used to sign them?

---------

### Answers:

1. Lots of algorithms are available:
    RSAMD5
    RSASHA1
    RSASHA1-NSEC3-SHA1
    RSASHA256
    RSASHA512
    ECC-GOST
    ECDSAP256SHA256
    ECDSAP384SHA384
    DSA
    DSA-NSEC3-SHA1
    hmac-md5.sig-alg.reg.int
    hmac-sha1
    hmac-sha256
    hmac-sha224
    hmac-sha384
    hmac-sha512 
    
    I prefer not to use MD5 and SHA1 specifically because they are broken and not secure enough. And because we need to sign every record with ZSK that should not be a long key. I prefer to use RSASHA256 cause it has enough security and speed to use.
    
2. Zone Signing Key (ZSK) is used to protect all zone data, and a Key Signing Key (KSK) is used to protect the zone’s keys (It signs the public ZSK (which is stored in a DNSKEY record), creating an RRSIG for the DNSKEY)
3. They are separated because it is more secure and easy to use. We can update them without dependency. For example, as I read, we need to update our ZSK more often, but KSK needs to be updated each year for example. And KSK is just encrypting the ZSK, so we will use longer keys to make it more secure (it is the beginning of this chain) but, for ZSK we need to consider shorter keys to be able to use it more often to sign each record.
4. For this part, I used the command below to create both KSK and ZSK.

```
ldns-keygen -a RSASHA256 -b 2048 -r /dev/urandom -k st5.sne21.ru

ldns-keygen -a RSASHA256 -b 1024 -r /dev/urandom st5.sne21.ru
```
Running these commands will create 5 new files. Two .private (private keys for ZSK and KSK), two .key (public keys for ZSK and KSK) and a DS file.

Then, I needed to sign the zone file using both private keys.

```
ldns-signzone st5.sne21.ru.zone Kst5.sne21.ru.+008+21438 Kst5.sne21.ru.+008+42196
```

After that, It was needed to tell nsd to use .zone.sign file that was created after running the command above.


![](https://i.imgur.com/LEQSb4g.png)
<p align=center>
<i>Figure 5: nsd confugration to use signed zone file</i>
</p>

![](https://i.imgur.com/b19fBwj.png)
<p align=center>
<i>Figure 6: Setup validation.</i>
</p>

5. Yes. My parent zone has my DS record which is necessary for the process of creating the secure chain and is used to say that my zone is authoritative.
6. DS record contains the hash of a DNSKEY record. And, DNSKEY Contains a public signing key (ZSK). The DNSKEY is used to sign the zone and when the resolver reaches my zone it will ask my parent for my zone's DS record and ask me about the DNSKEY. And will use these two to validate my zone records. Besides, DNSKEY will be signed by KSK.

----------
## Task 4: Key Rollovers

Zone-Signing Key rollover
1. What are the options for doing a ZSK rollover? Choose one procedure and motivate your choice
2. How would you integrate this procedure with the tools for signing your zone? Which timers are important?

Key-Signing Key rollover

3. Can you use the same procedure for a KSK rollover?
4. What does this depend on?

---------------

### Answers:

1. * Pre-Publication: A new key is introduced into the DNSKEY RRset, which is then re-signed.  This state of affairs remains in place for long enough to ensure that any cached DNSKEY RRsets contain both keys.  At that point, signatures created with the old key can be replaced by those created with the new key.

    * Double-Signature: this involves introducing the new key into the zone and using it to create additional RRSIG records; the old key and existing RRSIG records are retained.
    * Double-RRSIG: A true Double-Signature
      method (here called the Double-RRSIG method) involves introducing new signatures in the zone (while still retaining the old ones) but not introducing the new key.
      
  I want to use Pre-Publication because it does not increase the size of the zone and the size of responses.
  
2. First, I need to create a new ZSK and edit the zone, raise the serial and add the content of the new key to the zone. Then, Re-signing the zone with old keys. Now, publish the new zone with NSD and wait for all caching servers to update. After that, edit the zone and raise serial again. This time we'll use the new ZSK for signing, and include the old one as a passive key. To do this, we need to add the content of the old key to the zonefile as well.Then,Re-signing the zone, using the new key.
 For the clean up phase: Edit the zonefile, raise the serial and remove both .key contents from the file. From this point on, the old key files are no longer needed.
 
4. & 4.  Rolling the KSK requires interaction with the parent zone. And the method is a bit different from ZSK rollover and is called Double signature (Double-KSK) and after that, we need to send the new DS record for our parent
 
 
---
## References:
1. [Cache Poisoning Attack](https://www.sciencedirect.com/topics/computer-science/cache-poisoning-attack)
2. [DNS cache poisoning-History](https://arstechnica.com/information-technology/2020/11/researchers-find-way-to-revive-kaminskys-2008-dns-cache-poisoning-attack/)
3. [What is DNS cache poisoning](https://www.cloudflare.com/learning/dns/dns-cache-poisoning/)
4. [What is DNS Cache Poisoning and DNS Spoofing](https://www.kaspersky.com/resource-center/definitions/dns)
5. [DNS Spoofing](https://www.imperva.com/learn/application-security/dns-spoofing/)
6. [Island of Trust](https://www.techrepublic.com/blog/it-security/you-dont-have-to-wait-to-deploy-dnssec/)
7. [DNS over TLS vs. DNS over HTTPS](https://www.cloudflare.com/learning/dns/dns-over-tls/)
8. [DoH Isn’t Better](https://www.dnsfilter.com/blog/dns-over-tls)
9. [How To Set Up DNSSEC on an NSD Nameserver](https://www.digitalocean.com/community/tutorials/how-to-set-up-dnssec-on-an-nsd-nameserver-on-ubuntu-14-04)
10. [**Dnssec howto with NSD and ldns](https://www.whyscream.net/wiki/Dnssec_howto_with_NSD_and_ldns.md)
11. [Complete information about DNSSEC](https://bind9.readthedocs.io/en/latest/dnssec-guide.html#signing-verification)
