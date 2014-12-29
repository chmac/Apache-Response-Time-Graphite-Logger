NOTE: see https://github.com/manos/varnish-statsd-graphite-logger for a more evolved example.
It uses inotify to watch a file, rather than potentially blocking a web server in Logging.

## Summary
Python script designed to run from an Apache CustomLog pipe configuration, which reads stdin (a log line) and sends data to graphite via statsd.

Statsd is used, so it's, in theory, non-blocking UDP..
This logging daemon uses ~5MB and has been tested with >500 req/s sustained, per server. 

It logs: 1) response-time per URI, and 2) requests per second, per top-level URI and for an entire cluster of like-named servers.

This is mostly for example sake: you should be logging your web server's view of request-time, in addition to application-level views. Furthermore, it's awesome to use graphite for req/s stats, because parsing web server logs at high scale is insanity. 

We often found ourselves trying to determine why application-level response time metrics (sent from a tornado service to graphite) were great, but the actual request time from the user perspective was not as good. More data is better, so this script logs apache's view of the request-time... which is often drastically different from the application-level view.

There is a lot of site-specific stuff in here (like cluster_name), but it's still useful and well-documented. Adjust to taste.

## Configuration
For the most simple configuration, simply create a LogFormat in apache:

    LogFormat '%U %D' logformat_apache_timers

And then add a CustomLog line in your vhosts:

    CustomLog "|/path/to/script --cluster=arbitrary_grouping_name logformat_apache_timers 

Done! Apache will run the script on startup and start sending logs to it.


Krux configures a LogFormat in apache as: 

	LogFormat '%U %D' logformat_apache_timers

And then, in a vhost template via puppet (so every site/vhost gets it):

    <% if apache_timers %>
           CustomLog "|/path/to/script --cluster=<%= cluster_name -%>" logformat_apache_timers env=apache_statsd_logging
    <% end %>

And the final bit to glue it together, is setting the env variable (apache_statsd_logging) within a Location directive. We do this because many of our vhosts are serving different apps via Location directives (snippets generated by puppet), so we simply SetEnv in each Location, so we're only logging "endpoints" (the first part of the URI after the /) that we've configured. This maps nicely to "apps" for us. When we first started using this, we got so many metrics per second that graphite scaling was going to be our next project (over 200K/s requires serious graphite work) - due to random bots on the internet hitting URIs like /phpmyadmin, etc.. We converted to only known URLs, and drastically cut back on logging.

If you want to log ALL urls, change the script to send statsd data for endpoint_full.