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

A simple message from `1.555.555.5555` with the text "Hello world!" would be converted to a GET request to `http://mysite.com?msisdn=15555555555&to=[my number]&messageId=000000FFFB0356D1&text=Hello+world!&type=text&message-timestamp=2015-07-30+20%3A38%3A23`.

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

## The Final Application

The application, while simple in its end goal, was comprised of multiple moving parts.

First, I invested in an inexpensive, pre-paid internation phone with which I could send SMS messages while I was out. 

Second, I used Nexmo to configure a phone number in Sweden so I could retrieve and interact with any messages programmatically.

Unfortunately, I couldn't push data _directly_ from Nexmo to WordPress.com. Instead, I wrote a quick WordPress plugin on a server I managed in Toronto that would intercept the requests from Nexmo and convert any messages into a format appropriate for WordPress.

SMS messages sent from my phone to Nexmo [were turned into GET requests](https://docs.nexmo.com/index.php/sms-api/handle-inbound-message) populating, among other fields, a `text` query argument. All I had to do was listen for and handle any GETs that appeared to come from Nexmo.

```php
/**
 * Enable a /nexmopost endpoint on the site. This means any requests for
 * http://site.url/nexmopost will set the `nexmopost` query arg to `true`
 * within WordPress.
 */
function add_nexmo_endpoint() {
	add_rewrite_endpoint( 'nexmopost', EP_ALL );
}
add_action( 'init', 'add_nexmo_endpoint' );

/**
 * If the `nexmopost` query arg is set, don't display a regular template
 * file. Instead, sanitize the content and pass it along to our other site.
 */
function nexmo_template_redirect() {
    global $wp_query;
    if ( ! isset( $wp_query->query_vars['nexmopost'] ) ) return;
    
    $message_text = isset( $_REQUEST['text'] ) ? $_REQUEST['text'] : '';
    
    // Run our sanitization filters to make sure it's an OK request
	$message_text = apply_filters( 'content_save_pre', $message_text );
	
	if ( empty( $message_text ) ) die();
	
	$clien = new WP_HTTP_IXR_CLIENT( 'http://site.url/xmlrpc.php' );
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

This plugin stored a copy of every message it received in the WordPress database. Then it used the ubiquitous and reliable XML-RPC interface built into WordPress to push messages across the wire to my free blog at WordPress.com.

![First SMS Message on WordPress.com](https://s3-us-west-2.amazonaws.com/6675d06c-ea96-49d2-8788-c5bc5129fb4a/first_message.png)

In all, my updates travelled:
- From a feature phone in Haiti
- To an SMS gateway in Sweden
- To a VPS in Canada
- To a data center in the US
- To my Facebook wall so everyone could play along

It was a _lot_ of engineering to do something very simple - send a message to my friends to let them know I was OK.

## Looking Forward...

About 10 years ago, integrating a website with an SMS gateway was technically difficult costly, and legally complicated. You needed to have a partnership with an actual phone company and, often, any numbers you had available for programmatic use had to serve double-duty as either a phone or a fax line.

In short: it was a pain.

Today, though, Nexmo and Twilio are leading a pack of _multiple_ gateway providers that present developers with easy-to-use APIs for service integration. One tool might ingest SMS messages from customers reviewing products, displaying those reviews on a site. Another tool my generate _outbound_ messages intended to alert customers to events on a website (payments due, meeting reminders, comment notifications, etc).

The challenge today isn't in partnering with a gateway provider, it's in keeping your imagination from running wild as you start to work on a project.
