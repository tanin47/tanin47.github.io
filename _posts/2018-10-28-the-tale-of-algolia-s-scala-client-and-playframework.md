---
layout: post
title: "The tale of Algolia's Scala client and Playframework"
date: 2018-10-28
category: technical
---

Yesterday I've spent 5 hours debugging a strange issue when using the Algolia's Scala client with Playframework during local development. That is, the period where auto-reloading happens when code is changed.

For a long time (~9 months), we've experienced a DNS error "flakily"; later, it turned out to be a non-flaky issue. But more on that later.

The error is an injection error. We can't initialize `AlgoliaClient` because we can't initialize `DnsNameResolver`.

```
Caused by: java.lang.NullPointerException
  at io.netty.resolver.dns.DnsNameResolver.<init>(DnsNameResolver.java:303)
  at io.netty.resolver.dns.DnsNameResolverBuilder.build(DnsNameResolverBuilder.java:379)
  at algolia.AlgoliaHttpClient.<init>(AlgoliaHttpClient.scala:56)
  at algolia.AlgoliaClient.<init>(AlgoliaClient.scala:64)
...
```

Please note that, for a newer version of Netty, the error will be around `FailedChannel` cannot be casted to Channel. But it still occurs at the same place.

Well, ok, it was a network thingy. It could be flaky, so I ignored it for several months.

Yesterday, I've found out that this injection error happens almost exactly when Playframework reloaded code around 23 times. It took me a lot of times to test this hypothesis because changing code 23 times was tedious.

I wrote a script that instantiating `AlgoliaClient` multiple times, and the exception was always raised at the 26rd `AlgoliaClient`.  The exception wasn't exactly helpful though. I did a lot of random things afterward with no progress for another hour.

What helped me progress was that I tried to instantiate the 27st `AlgoliaClient`, and there it was. The actual exception showed itself:

```
Caused by: java.net.SocketException: maximum number of DatagramSockets reached
  at sun.net.ResourceManager.beforeUdpCreate(ResourceManager.java:72)
  at java.net.AbstractPlainDatagramSocketImpl.create(AbstractPlainDatagramSocketImpl.java:69)
  at java.net.TwoStacksPlainDatagramSocketImpl.create(TwoStacksPlainDatagramSocketImpl.java:70)
```

After googling for [ResourceManager](https://github.com/JetBrains/jdk8u_jdk/blob/master/src/share/classes/sun/net/ResourceManager.java#L73), it turned out that the limit to the number of datagram sockets was 25!

I tested this hypothesis by running `sbt Dsun.net.maxDatagramSockets=4 `runMain TheScript'`, and the script failed after the 4st `AlgoliaClient`.

Now I know that the instantiated `AlgoliaClient` wasn't cleaned up properly, so I simply needed to close it.

Unfortunately, `algoliaClient.close()` [doesn't close its DNS name resolver](https://github.com/algolia/algoliasearch-client-scala/issues/500). Fortunately, the DNS name resolver is public, and I can close it myself with `algoliaClient.httpClient.dnsNameResolver.close()`.

Now I knew I needed to clean up `AlgoliaClient` before Playframework reloads code. And, fortunately, Playframework offers [a stop hook](https://www.playframework.com/documentation/2.6.x/ScalaDependencyInjection#Stopping/cleaning-up) that is invoked before code reloading.

Here are some lessons for me:

* I should have wrote the script much earlier. Changing code 25 times was extremely tedious and time-wasting. Reproducing a bug easily would have saved me a lot of time.
* Biased to make class's members public. It'll be helpful for future users. Scala gets it right to default everything to public, while Java defaults everything to somewhat private. Making a member private should be a conscious and thoughtful decision. If DNS name resolver wasn't public, it would be tricky for me to progress.
* I should have not ignored the injection error during local development, no matter how flaky it is.


