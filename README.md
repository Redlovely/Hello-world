# Hello-world
My first try

I don't have chose .. what going on ? 



### The perfect Varnish configuration for Joomla, WordPress & other CMS based websites ###
##########################################################################################

IMPORTANT: Read this before implementing one of the configuration files below (for either Varnish 3.x or 4.x+).

USE: Replace the contents of the main Varnish configuration file located in /etc/varnish/default.vcl
(root server access required - obviously) with the contents of the configuration you'll use (depending on your Varnish version)
from the 2 examples provided below.

IMPORTANT: The following setup assumes a 2 minute cache time. You can safely increase
this to 5 mins for less busier sites or drop it to 1 min or even 30s for high traffic sites.

This configuration requires an HTTP Header and a user cookie (see the Joomla section)
to identify if a user is logged in a site, in order to bypass caching overall. If your CMS provides a way to add
these 2 requirements, then you can use this configuration to speed up your site or entire server. You can even
exclude the domains you don't want to cache if you're looking to use it in a multi-site setup.



=== JOOMLA & VARNISH ===
Since Joomla v3.6, all you need to do to have Joomla play nicely with Varnish is add your exclusion points (URLs)
- see the 2 blocks starting with "Exclude the following paths..." below in the configuration.

If you're using Joomla before version 3.6, you need to do the following:
This Varnish configuration makes use of a custom HTTP header plus a user cookie to determine whether
some user is logged in or not inside Joomla. To insert the HTTP header, simply append the following code block
in your template's "index.php" file, right after the line:
defined('_JEXEC') or die;
...and make sure you set the $cookieDomain value:

// Make Joomla Varnish-friendly [START]
$cookieDomain = 'domain.tld'; // Replace "domain.tld" with your "naked" domain

$getUserState = JFactory::getUser();

if ($getUserState->guest) {
    JResponse::allowCache(true);
    JResponse::setHeader('X-Logged-In', 'False', true);
    if($_COOKIE["userID"]){
        setcookie("userID", "", time() - 3600, '/', $cookieDomain, 0);
    }
} else {
    JResponse::allowCache(true);
    JResponse::setHeader('X-Logged-In', 'True', true);
    if(!isset($_COOKIE["userID"])){
        setcookie("userID", $getUserState->id, 0, '/', $cookieDomain, 0);
    }
}
// Make Joomla Varnish-friendly [FINISH]

IMPORTANT: If you use K2 (getk2.org) in your Joomla site, simply set the "Cookie Domain" option in the K2 parameters
("Advanced" tab) and all the above will be automatically enabled for your entire Joomla site.



=== HOW TO HANDLE FRONTEND LOGINS (e.g. for use with member areas, forums etc.) ===
It is important for you to understand that since Joomla (in a very amateur way) uses session cookies for any user
(even guests) supposedly for additional security (debatable), Varnish *cannot* work with Joomla out-of-the-box. If
you installed Varnish without any modification to its configuration besides the cache time, it could not properly
cache Joomla content because of the session cookies Joomla uses for both guest and logged in visitors. To bypass
Joomla's behaviour, we must additionally set Varnish to strip any cookies set by Joomla, except for a specific one (userID).
For even better control, we also set a custom HTTP header (X-Logged-In), which we have Varnish check on all requests. All
this is explained how to integrate into Joomla via your template in the code sample above.
However, if we want Varnish to allow frontend logins in Joomla, without breaking Joomla (because we strip its session cookies),
we must explicitly tell Varnish which entry pages (=login pages) not to cache. Such a page could be for example the default
Joomla login form (e.g. with an alias "login"). In the 2 Varnish exclusion lists defined in the configuration below, we would add
"^/login" to make sure Varnish completely switches off when a user visits this page. In that case, Joomla's session cookie gets
set and the form can be submitted normally, passing all Joomla security checks. Same goes for any page in Joomla that requires
user input: a contact form, a newsletter signup form, a forum, comments and so on. So the solution to keep in mind is simple:
- If the action requires the user to login first (e.g. a forum), we must create a specific/unique page for users to login first.
  Once they log in, Varnish switches off completely and then a user can post in the forum or write comments or use a contact form
  as if Varnish did not exist. If the user continues to browse the site while logged in, Varnish will be completely off ONLY for
  this user. If the user logs out, Varnish will kick back in.
- If the action does not require a user to be logged in first, e.g. a contact form, we simply exclude the contact form's URL from
  Varnish, in which case -again- Varnish will switch off completely and the user will be able to submit the form passing the
  Joomla security checks. If the user browses anywhere else in the site, Varnish will kick back in.
