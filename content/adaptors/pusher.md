# Ably Pusher protocol adaptor

**Our protocol adaptors are in public beta. If you wish to use our adaptors, please [contact us first](https://www.ably.io/contact) using our live chat or email support@ably.io**

To use the Ably Pusher protocol adaptor, you must initialize your Pusher client library as follows, assuming an [Ably API key](https://support.ably.io/solution/articles/3000030054-what-is-an-app-api-key) of `'appid.keyid:keysecret'`:

```js
var pusher = new Pusher('appid.keyid', {
  wsHost       : 'realtime-pusher.ably.io',
  httpHost     : 'realtime-pusher.ably.io',
  disableStats : true,
  encrypted    : true
});
```

And your Pusher REST libraries as (using ruby as an example):

```ruby
Pusher::Client.new(
  app_id:    'appid',
  key:       'appid.keyid',
  secret:    'keysecret',
  host:      'rest-pusher.ably.io',
  encrypted: true
)
```

[View a Pusher adaptor example that receives messages in your browser](<%= JsBins.url_for('adaptors/pusher-pub-sub') %>)

Please note:

* For simplicity, the above example uses the Pusher Javascript and Ruby libraries. All other Pusher client libraries can be instanced in a similar fashion. See the [complete list of Pusher client libraries and their documentation](https://pusher.com/docs/libraries).
* For the Pusher key, use the part of the [Ably API key](https://support.ably.io/solution/articles/3000030054-what-is-an-app-api-key) before the colon; for the secret, use the part after the colon.
* You can add any other Pusher options you would normally use in the initializer.
* In the Pusher client library terminology, the 'encrypted' option controls whether the the lib uses TLS, not end-to-end encryption. Setting the `encrypted` option to true is not mandatory, but strongly recommended to avoid sending private keys over plain text connections.
* Pusher adaptor clients are not "identified" in the Ably sense. Make sure you do not have 'require identification' channel rule on for any channels/namespaces you will be accessing with the adaptor. See [this article](https://support.ably.io/support/solutions/articles/3000030057-what-are-channel-rules-and-how-can-i-use-them-in-my-app-) on how to change channel rules. If you get an "Ably error 40160", this is the reason why.
* *The Pusher channel will have a different name from the corresponding Ably channel*. See the 'Channels' section below.

## Supported features

- REST publish
- REST get occupied channels (*Note: channel enumeration is in early alpha, and is not ready for production use. You may experience timeouts when using this feature; this is a known issue*)
- REST get presence set
- REST get user count
- Realtime subscribe
- Realtime publish (ie client events)
- Realtime presence

## Unsupported features

- REST subscriber count (partially supported; see below)

## Usage notes

### General

- While the Pusher adaptor is quite a light translation layer (certainly a good deal lighter than the Pubnub adaptor), a protocol adaptor inevitably adds some latency. Using the adaptor will be a little slower than using Ably native client libraries. Typically the impact is in the low milliseconds. By the same token, it will likely also be marginally slower than using Pusher natively -- but only if you are close to whichever Pusher datacenter you are using. If not, the extra milliseconds of latency from the adaptor should be more than compensated for by being able to use a datacenter closer to you (unlike Pusher, Ably has datacenters in [9+ regions worldwide](https://support.ably.io/solution/articles/3000029525-where-are-ably-s-servers-located-around-the-world) and federates messages between them - you automatically connect to the closest one to you, using latency-based routing).
- Behind the scenes, the adaptor just uses the normal Ably service, so there is no problem with using Pusher and Ably client libraries side by side (though bear in mind channel name translation, see below). You can mix and match as you like; for example, using Pusher client libraries normally, but using the Ably REST api to get channel history, which is not available through the Pusher clients.
- While using the adaptor gives you some of the advantages of Ably over Pusher (eg inter-region message federation), many others (e.g. [continuity guarantees](https://support.ably.io/solution/articles/3000044639-connection-state-recovery), [fallback host support](https://support.ably.io/solution/articles/3000044636-routing-around-network-and-dns-issues), [history](https://www.ably.io/documentation/realtime/history), [flexible channel namespaces](https://support.ably.io/solution/articles/3000030058-what-is-a-channel-namespace-and-how-can-i-use-them-), [powerful token authentication](https://www.ably.io/documentation/general/authentication)) require the use of the Ably client libraries. As a result, if a native Ably library is available for your platform, we recommend you consider using the Ably client libraries instead or at least make a plan to eventually transition over to Ably native client libraries.
- Please note that that channel enumeration (the /channels REST endpoint) is in early alpha, and is not ready for production use. You may experience timeouts when using this feature; this is a known issue.

### Security

When using Ably normally, you cannot connect to the Ably service at all without a complete API key, or a token derived from one. When using the Pusher adaptor, you can connect to public channels (see 'Channels' below) with only the Ably key name (the app id together with the key id). Since the Pusher adaptor lets you do something that would not normally be possible with Ably, it needs to be explicitly enabled for each api key you want to use it with. If you would like it enabled, please [contact us through live chat](https://www.ably.io/contact) or by emailing support@ably.io.

### Channels

- Ably and Pusher both have various (incompatible) restrictions on channel names, and use namespaces very differently. So when using the adaptor, channel names are mapped. That is, if you connect to the channel "foo" using the Pusher adaptor, it will actually use the Ably channel "public:foo". The channel name translation rules are as follows:
 - Pusher public channels (no prefix) are mapped to the the Ably 'public:' namespace.
 - Pusher private channels ('private-' prefix) are mapped to the the Ably 'private:' namespace.
 - Pusher presence channels ('presence-' prefix) are mapped to the the Ably 'presence:' namespace.
 - Conversely, Ably channels not in any of the 'public:', 'private:', or 'presence:' namespaces get a 'private-ablyroot-' prefix.
 - Semicolons are banned in Ably channel names; colons are banned in Pusher channel names. So the adaptor maps one to the other: semicolons in Pusher channel names become colons in Ably channel names.

A few examples:

Pusher adaptor channel    | Actual Ably channel
--------------------------|-------------
presence-foo              |  presence:foo
private-foo               |  private:foo
foo                       |  public:foo
foo;bar                   |  public:foo:bar
private-ablyroot-foo      |  foo
private-ablyroot-foo;bar  |  foo:bar

### User and subscriber count

Getting total subscriber count for a channel is a planned feature in Ably. For now, 'subscriber count', like 'user count', will only work with presence channels. 'User count' gives the size of the presence set once uniqified by [`clientId`](https://www.ably.io/documentation/realtime/authentication#identified-clients), and 'subscriber count' gives the raw size of the presence set.
