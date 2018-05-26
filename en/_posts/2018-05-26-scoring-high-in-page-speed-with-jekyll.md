---
layout: post
title: Scoring high in PageSpeed with Jekyll
lang: en
lang\_order: 2
ref: scoring-high-in-page-speed-with-jekyll-and-nginx
--- 

![][image-1]

## Use something simple

I built this site using [Jekyll][1] because I wanted something simple enough for:

1. Focusing on the content and not site administration (updates, plugins, security and that kind of mandatory stuff for some popular CMS).
2. Building something fast, not over bloated with database connections, Javascript, CSS and calls to external sites.

So I chose a simple Jekyll template, customised it, initialized the Jekyll project, deployed it to a Google Cloud Engine VPS and run the PageSpeed Insights tool on it, because one of the first things you do when you put a new site online is to check its performance using Google PageSpeed Insights tool or similar, right?

## Test it

The test scored terrible, about a 70% for a page with little CSS, just one image, simple HTML, the minimum JS to report to Google Analytics and just a few paragraphs of content.

Among several problems, these appeared to be the main ones:

- External calls (Google Analytics and Google Fonts).
- Non minimized CSS/HTML.
- Non-optimized images.
- Missing cache headers.

Really, really disappointing. 

## Fix the problems

Ok,  the image had a size than more than 5 megabytes, that’s way too much. I used an online service for optimizing the png file, and it went down to 1,2 megabytes. 

I also minimized the CSS and downloaded the fonts to serve them statically.

That should suffice I thought...

However, I still scored \< 80%, both mobile and desktop versions, for a straightforward simple static blog. WTF.

## Actually fix the problems

I got serious and really investigated how could I fix the *performance* problems PageSpeed had reported. My advantage, I was serving the site from a custom VPS server, so I had full control over it.

Moreover, I discovered **[Apache PageSpeed][2]**,  open-source modules that optimize your site automatically. Too good to be true.

They provide modules for both Apache and Nginx servers. The only caveat, I had to uninstall Nginx from the CentOS repository and download and build it from source to compile it with the pagespeed modules. An easy task as they provide an [automated install][3] script (just like rvm!).

They provide a ton of [mods][4]! Let’s try to keep it simple.

What did I need:
- Image optimizations.
- Make sure Google Analytics is called asynchronously.
- Minimize HTML and CSS.

These are the key PageSpeed modules:

- make_google_analytics_async;
- collapse_whitespace,combine_css,rewrite_css,move_css_above_scripts,combine_javascript,rewrite_javascript;
- lazyload_images,inline_preview_images,resize_mobile_images,responsive_images,resize_images,rewrite_images;

And this is the resulted Nginx configuration file for the site:

	server {
	  server_name yanez.io;
	
	  root /home/web/current;
	  index index.html;
	
	  error_log /var/log/nginx/yanez.io.error.log;
	  access_log  /var/log/nginx/yanez.io.access.log;
	
	  listen 443 ssl http2; # managed by Certbot
	  ssl_certificate /etc/letsencrypt/live/yanez.io/fullchain.pem; # managed by Certbot
	  ssl_certificate_key /etc/letsencrypt/live/yanez.io/privkey.pem; # managed by Certbot
	  include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
	  ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
	
	  pagespeed on;
	
	  # Needs to exist and be writable by nginx.  Use tmpfs for best performance.
	  pagespeed FileCachePath /var/ngx_pagespeed_cache;
	
	  # Ensure requests for pagespeed optimized resources go to the pagespeed handler
	  # and no extraneous headers get set.
	  location ~ "\.pagespeed\.([a-z]\.)?[a-z]{2}\.[^.]{10}\.[^.]+" {
	    add_header "" "";
	  }
	  location ~ "^/pagespeed_static/" { }
	  location ~ "^/ngx_pagespeed_beacon$" { }
	
	  pagespeed EnableFilters make_google_analytics_async;
	  pagespeed EnableFilters collapse_whitespace,combine_css,rewrite_css,move_css_above_scripts,combine_javascript,rewrite_javascript; 
	  pagespeed EnableFilters lazyload_images,inline_preview_images,resize_mobile_images,responsive_images,resize_images,rewrite_images;
	
	  if (!-f "${request_filename}index.html") {
	    rewrite ^/(.*)/$ /$1 permanent;
	  }
	
	  if ($request_uri ~* "/index.html") {
	    rewrite (?i)^(.*)index\.html$ $1 permanent;
	  }
	
	  if ($request_uri ~* ".html") {
	    rewrite (?i)^(.*)/(.*)\.html $1/$2 permanent;
	  }
	
	  location ~*  \.(jpg|jpeg|png|gif|ico|css|js|pdf|woff2)$ {
	    expires 30d;
	  }
	
	  error_page 404 /404.html;
	
	  location / {
	    try_files $uri $uri.html $uri/ =404;
	  }
	}

So, without changes to the site code, just adding PageSpeed module, I finally scored **good** in Google PageSpeed Insights:


## Some thoughts

Not sure about the utility of PageSpeed. It looks like the important thing is the score and not the actual user experience.

For some stuff it's useful. For example, I didn't notice I put an excessively large image, and it pointed it out. Also, using the server's cache for static resources is basic and it has a dedicated test for it.

However, will the user notice an 8.6kb CSS file is not minimized?  It would be handy it PageSpeed told you what's the impact in the score of every issue it detects.

Some other paid tools like [WooRank][5] splits the issues it detects in different categories, like issues that would be good to correct, and issues that you must correct. But PageSpeed directly puts all the issues at the same level:

![][image-2]

If you are using PageSpeed, you are a technical user. You don't need complexity to be hidden. You want to know the details. You want to know that an 8.6kb CSS file being minimized would have an impact of 1.5% in your score and activate the cache in your server a 10%.

Anyway, despite all this simplicity, and being free, PageSpeed is a tool I'll keep using. It's from Google, it scores your site, and if your site does not score well in Google is a bad thing, right?










[1]:	https://jekyllrb.com "Jekyll"
[2]:	https://www.modpagespeed.com "Apache PageSpeed"
[3]:	https://www.modpagespeed.com/doc/build_ngx_pagespeed_from_source "automated install"
[4]:	https://www.modpagespeed.com/doc/ "mods"
[5]:	https://www.woorank.com "WooRank"

[image-1]:	/assets/images/2018-05-26_22-30-37.png "Google PageSpeed Insights"
[image-2]:	/assets/images/2018-05-26_22-29-49.png "PageSpeed results"