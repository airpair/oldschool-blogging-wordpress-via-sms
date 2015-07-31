In late 2012, I was presented with an impromptu opportunity to visit the island nation of Haiti. The trip was an exciting chance to see the world, and I wanted to keep my friends and family updated while I traveled.

I needed a quick, reliable way to push out information to several people at once. My day job involves building [WordPress](http://wordpress.org), so the first thing I did was set up a blog on [WordPress.com](http://wordpress.com). I also integrated it with Facebook so none of my friends had to go hunting for a URL.

But using a website to dispense information raised another problem: **How would I keep things updated when traveling in the mountains of Haiti?**

## Rapid Prototyping

I had been an avid Twitter user in the days when its primary interface was SMS messages to and from 40404. It was a good enough system for my less-than-smart college phone; I figured it might be a good solution for Haiti, too!

My first proof-of-concept prototype used an [IFTTT](https://ifttt.com/) recipe to automatically push content from Twitter into my blog on WordPress.com. It worked like a dream!

I just knew it wasn't sustainable as Twitter was killing the IFTTT integration a few days before I left. Still, it was proof that a service could consume SMS messages and convert them to WordPress blog posts.

## Beyond a Prototype

To remotely publish my articles, I needed two components:
- A service to receive SMS notifications from my international phone
- A WordPress plugin to convert these notifications into blog posts

The service was the easiest part. I opened an account with [Nexmo](https://www.nexmo.com), a newer SMS gateway at the time. Nexmo provided me with a phone number in Sweden I could easily text from Haiti. It also took any messages received from my new number and converted them to a web request of whatever server I wanted.

A simple message from `1.555.555.5555` with the text "Hello world!" [would be converted to a GET request](https://docs.nexmo.com/index.php/sms-api/handle-inbound-message) to `http://mysite.com?msisdn=15555555555&to=[my number]&messageId=000000FFFB0356D1&text=Hello+world!&type=text&message-timestamp=2015-07-30+20%3A38%3A23`.

Nexmo did all of the heavy lifting for me. It logs the message as being received, assigns a unique identifier to it, and dispatches a GET to the server of my choice.

All I had to do at this point was get WordPress to understand the message.

## WordPress, WordPress.com, and Plugins

[WordPress.com](https://wordpress.com), the free, _hosted_ blogging platform, doesn't support custom plugins. To get my posts from Nexmo into my free blog, I had to set up a _second_ WordPress site as an intermediary.

This secondary site lived on a server in Canada and did little more than parse incoming Nexmo requests and convert them into outgoing XML-RPC routines.

### Custom Plugin

WordPress plugins empower developers to do whatever they want on the platform. The action/filter hook system lets you set up custom code that fires in specific places and completely breaks the normal flow of WordPress execution.

Natively, WordPress allows developers to register custom endpoints - strings affixed to the end of a URL that trigger the population of custom variables. In this case, I needed the site to listen for any GET requests to a `/nexmo` endpoint and handle them properly.

```php
/**
 * Enable a /nexmo endpoint on the site.
 */
function add_nexmo_endpoint() {
	add_rewrite_endpoint( 'nexmo', EP_ALL );
}
add_action( 'init', 'add_nexmo_endpoint' );
```

These three lines of code tell WordPress that, if any URL ends in `/nexmo` to automatically populate the value of `$_REQUEST['nexmo']` with `1`. This flag allows us to hook in before WordPress generates any templates or runs any other code.

Any requests sent from Nexmo itself to `http://mysite.com/nexmo` will automatically trigger this code and be intercepted later down the chain.

### XML-RPC

XML-RPC is one of the ugliest API implementations in the world; it's also one of the most reliable. Through XML-RPC you can control nearly every aspect of a WordPress blog - creating, editing, or deleting content, updating settings, you name it.

To create a post in WordPress over XML-RPC, you POST an XML document to `http://mysite.com/xmlrpc.php`. The document details the method you want invoked and the data you want to invoke it with.

This XML document would, for example, create a basic "Hello World!" post in the default category on a blog:

```xml
<?xml version="1.0"?>
<methodCall>
  <methodName>wp.newPost</methodName>
  <params>
    <param>
      <value><int>0</int></value>
      <value><string>username</string></value>
      <value><string>password</string></value>
      <value>
        <struct>
          <member>
            <name>post_status</name>
            <value><string>publish</string></value>
          </member>
          <member>
            <name>post_title</name>
            <value><string>Hello World!</string></value>
          </member>
          <member>
            <name>post_content</name>
            <value><string>This is some post content</string></value>
          </member>
        </struct>
      </value>
    </param>
  </params>
</methodCall>
```

It's not very pretty and would be a mess to build manually within _any_ coding system. Luckily, XML-RPC is built into WordPress, both in terms of a _server_ and as a _client_.

WordPress natively understands this XML document, will parse it, validate the arguments, and generate a post in the database as a result. Even better, WordPress includes a native XML-RPC client for _generating_ these kinds of messages without giving you a headache.

When our WordPress site sees the Nexmo flag above, we can parse the other information included in the `$_REQUEST` array and generate an XML-RPC post to the WordPress.com site where we want the content to live.

Another action call, hooking in to WordPress' `template_redirect` hook, allows us to parse and dispatch the message _before_ WordPress loads much of our theme or template files. It's lightweight and effective:

```php
/**
 * If the `nexmopost` query arg is set, don't display a regular template
 * file. Instead, sanitize the content and pass it along to our other site.
 */
function nexmo_template_redirect() {
    global $wp_query;
    if ( ! isset( $wp_query->query_vars['nexmo'] ) ) {
        return;
    }
    
    $message_text = isset( $_REQUEST['text'] ) ? $_REQUEST['text'] : '';
    
    // Run our sanitization filters to make sure it's an OK request
	$message_text = apply_filters( 'content_save_pre', $message_text );
	
	if ( empty( $message_text ) ) {
	    die();
	}
	
	$client = new WP_HTTP_IXR_Client( 'http://site.url/xmlrpc.php' );
	$result = $client->query( 'wp.newPost', array(
	    0,
	    'username',
	    'password',
	    array(
	        'post_status'  => 'publish',
	        'post_title'   => 'SMS Update',
	        'post_content' => stripslashes_deep( $message_text ),
	        'post_format'  => 'status',
	    )
	) );
}
add_action( 'template_redirect', 'nexmo_template_redirect' );
```

The [`WP_HTTP_IXR_Client` class](https://developer.wordpress.org/reference/classes/wp_http_ixr_client/) is a handy abstraction of the XML-RPC system built within WordPress. We use the class to connect to a remote XML-RPC server, and then query it by passing a method name and an array of arguments.
The first argument is (almost) always 0. This was meant to be a network identifier, but hasn't actually been implemented yet in any of WordPress' RPC calls.

Authenticated requests then take a WordPress username and password as their next arguments. **Note** that these are passed over the wire in plain text, so you should only really be using this on a server with proper SSL support.

The full documentation for the API and all available commands [is avaialable on the Codex](https://codex.wordpress.org/XML-RPC_WordPress_API).

## In Production

In all, my updates travelled:
- From a feature phone in Haiti
- To an SMS gateway in Sweden
- To a VPS in Canada
- To a data center in the US
- To my Facebook wall so everyone could play along

![First SMS Message on WordPress.com](https://s3-us-west-2.amazonaws.com/6675d06c-ea96-49d2-8788-c5bc5129fb4a/first_message.png)

It was a _lot_ of engineering to do something very simple - send a message to my friends to let them know I was OK.

## Looking Forward...

About 10 years ago, integrating a website with an SMS gateway was technically difficult costly, and legally complicated. You needed to have a partnership with an actual phone company and, often, any numbers you had available for programmatic use had to serve double-duty as either a phone or a fax line.

In short: it was a pain.

Today, though, Nexmo and Twilio are leading a pack of _multiple_ gateway providers that present developers with easy-to-use APIs for service integration. One tool might ingest SMS messages from customers reviewing products, displaying those reviews on a site. Another tool my generate _outbound_ messages intended to alert customers to events on a website (payments due, meeting reminders, comment notifications, etc).

The challenge today isn't in partnering with a gateway provider, it's in keeping your imagination from running wild as you start to work on a project.
