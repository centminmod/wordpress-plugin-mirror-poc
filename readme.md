# WordPress Plugin Mirror Downloader: User Documentation

## Table of Contents
1. [Introduction](#introduction)
2. [System Overview](#system-overview)
3. [Examples](#examples)
4. [Screenshots](#screenshots)

## Introduction

The WordPress Plugin Mirror Downloader is a sophisticated system designed to efficiently download, cache, and manage WordPress plugins. It leverages Cloudflare's edge computing capabilities and object storage to create a high-performance, scalable solution for plugin management.

### Key Components

1. **Cloudflare Worker**: A serverless JavaScript function that acts as an intermediary between the client (bash script) and the data sources (WordPress.org and Cloudflare R2 storage). See https://developers.cloudflare.com/workers/ and https://developers.cloudflare.com/workers/tutorials/. And how Cloudflare Workers have bindings to other Cloudflare products like Cloudflare R2 S3 object storage https://developers.cloudflare.com/workers/runtime-apis/bindings/.

2. **Cloudflare R2 Storage**: An S3-compatible object storage system used to cache plugin ZIP files and metadata JSON. Cloudflare R2 S3 object storage has free egress bandwidth costs so you only pay for object storage and read/writes to object storage. See Cloudflare R2 pricing https://developers.cloudflare.com/r2/platform/pricing/ and calculator at https://r2-calculator.cloudflare.com/.

Cloudflare R2 free plan quota pricing and PAYGO pricing beyond free plan.

| Feature | Forever Free | Standard Storage | Infrequent Access Storage (Beta) |
|---------|--------------|-------------------|----------------------------------|
| Storage | 10 GB / month | $0.015 / GB-month | $0.01 / GB-month |
| Class A operations: mutate state | 1,000,000 / month | $4.50 / million requests | $9.00 / million requests |
| Class B operations: read existing state | 10,000,000 / month | $0.36 / million requests | $0.90 / million requests |
| Data Retrieval (processing) | N/A | None | $0.01 / GB |
| Egress (data transfer to Internet) | N/A | Free | Free |

3. **Bash Script**: A local client that orchestrates the plugin download process, interacts with the WordPress API, and communicates with the Cloudflare Worker.

### System Advantages

- **Efficient Caching**: By utilizing Cloudflare R2 storage, the system significantly reduces the load on WordPress servers and improves download speeds for frequently requested plugins.

- **Version Tracking**: The system maintains a local record of installed plugin versions, enabling selective updates and reducing unnecessary downloads.

- **Parallel Processing**: The bash script supports concurrent downloads, dramatically reducing the time required for bulk plugin updates.

- **Comprehensive Logging**: Detailed logging at both the Worker and bash script levels facilitates troubleshooting and performance optimization.

- **Advanced Error Handling**: Robust error handling mechanisms in both the Worker and bash script ensure graceful failure recovery and informative error reporting.

- **Rate Limiting**: Implemented at the Worker level to prevent abuse and ensure fair usage of resources.

- **Compression Support**: The Worker supports gzip compression, reducing bandwidth usage and improving download speeds.

- **Separate Caching for ZIP and JSON**: By caching ZIP files and JSON metadata separately, the system can efficiently handle partial updates and reduce storage costs.

- **Cache-Only Mode**: A new feature allows checking and updating the cache and R2 bucket without downloading files, useful for preemptive caching and system checks.

## System Overview

The WordPress Plugin Mirror Downloader operates through a series of coordinated steps involving the bash script, Cloudflare Worker, and R2 storage. Here's a detailed breakdown of the process:

1. **Plugin Identification**:
   - The bash script reads a list of desired plugins from a configuration file or command-line arguments.
   - It compares this list against a local cache of installed plugins and their versions.
   - The script identifies plugins that need to be downloaded or updated based on version discrepancies.

2. **API Interaction**:
   - For each plugin requiring action, the script queries the WordPress.org API to fetch the latest version information and download URL.
   - This step ensures that the system always works with the most up-to-date plugin data.

3. **Worker Request Generation**:
   - The script constructs HTTP requests to the Cloudflare Worker for both the plugin ZIP file and its JSON metadata.
   - These requests include query parameters specifying the plugin name, version, and desired content type (ZIP or JSON).

4. **Worker Processing**:
   - Upon receiving a request, the Worker first checks if the requested data exists in the appropriate R2 bucket (WP_PLUGIN_STORE for ZIPs, WP_PLUGIN_INFO for JSONs).
   - If the data is found in R2, the Worker serves it directly, incrementing the `r2Hits` metric.
   - If not found, the Worker fetches the data from WordPress.org, stores it in R2, and then serves it. This process increments the `wordpressFetches` and `cacheMisses` metrics.

5. **R2 Storage Interaction**:
   - The Worker interacts with R2 storage using the `env.WP_PLUGIN_STORE` and `env.WP_PLUGIN_INFO` bindings.
   - For storage, the Worker uses `put` operations with appropriate metadata.
   - For retrieval, it uses `get` operations, falling back to WordPress.org if the object is not found.

6. **Data Compression**:
   - The Worker applies gzip compression to the response if the client supports it, as indicated by the `Accept-Encoding` header.
   - This compression is performed using the `CompressionStream` API.

7. **Response Handling**:
   - The bash script receives the Worker's response, which includes the requested data (ZIP or JSON) and relevant headers.
   - It processes this response, saving the data to the local filesystem and updating its version tracking information.

8. **Parallel Execution**:
   - When configured for parallel processing, the bash script uses `xargs` to spawn multiple instances of the download process.
   - This parallelization significantly reduces the total time required for bulk plugin updates.

9. **Logging and Metrics**:
   - Throughout the process, both the Worker and bash script maintain detailed logs.
   - The Worker tracks key metrics such as cache hits, WordPress fetches, and error counts.
   - The bash script logs each step of its operation, including version checks, download attempts, and file operations.

10. **Error Handling and Retries**:
    - The system implements multiple layers of error handling, including network retries in the Worker and fallback mechanisms in the bash script.
    - Detailed error messages are propagated from the Worker to the bash script, allowing for informed troubleshooting.

11. **Cache-Only Mode**:
    - When activated, this mode allows the system to check and update the cache and R2 bucket without downloading files.
    - It's useful for preemptive caching, system checks, and reducing unnecessary downloads.

This architecture allows for efficient, scalable, and resilient WordPress plugin management, leveraging the strengths of edge computing and distributed storage to create a robust mirroring system.

## Examples

```bash
time ./get_plugins_r2.sh -d

Processing plugin: advanced-custom-fields
[DEBUG] Checking latest version and download link for advanced-custom-fields
[DEBUG] Latest version for advanced-custom-fields: 6.3.6
[DEBUG] API download link for advanced-custom-fields: https://downloads.wordpress.org/plugin/advanced-custom-fields.6.3.6.zip
[DEBUG] Stored version for advanced-custom-fields: 6.3.6
advanced-custom-fields is up-to-date and exists in mirror directory. Skipping download...
[DEBUG] Saving plugin json metadata for advanced-custom-fields version 6.3.6
[DEBUG] json metadata for advanced-custom-fields version 6.3.6 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for advanced-custom-fields.
Processing plugin: akismet
[DEBUG] Checking latest version and download link for akismet
[DEBUG] Latest version for akismet: 5.3.3
[DEBUG] API download link for akismet: https://downloads.wordpress.org/plugin/akismet.5.3.3.zip
[DEBUG] Stored version for akismet: 5.3.3
akismet is up-to-date and exists in mirror directory. Skipping download...
[DEBUG] Saving plugin json metadata for akismet version 5.3.3
[DEBUG] json metadata for akismet version 5.3.3 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for akismet.
Processing plugin: amr-cron-manager
[DEBUG] Checking latest version and download link for amr-cron-manager
[DEBUG] Latest version for amr-cron-manager: 2.3
[DEBUG] API download link for amr-cron-manager: https://downloads.wordpress.org/plugin/amr-cron-manager.2.3.zip
[DEBUG] Stored version for amr-cron-manager: 2.3
amr-cron-manager is up-to-date and exists in mirror directory. Skipping download...
[DEBUG] Saving plugin json metadata for amr-cron-manager version 2.3
[DEBUG] json metadata for amr-cron-manager version 2.3 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for amr-cron-manager.
Processing plugin: assets-manager
[DEBUG] Checking latest version and download link for assets-manager
[DEBUG] Latest version for assets-manager: 1.0.2
[DEBUG] API download link for assets-manager: https://downloads.wordpress.org/plugin/assets-manager.zip
[DEBUG] Stored version for assets-manager: 1.0.2
assets-manager is up-to-date and exists in mirror directory. Skipping download...
[DEBUG] Saving plugin json metadata for assets-manager version 1.0.2
[DEBUG] json metadata for assets-manager version 1.0.2 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for assets-manager.
Processing plugin: autoptimize
[DEBUG] Checking latest version and download link for autoptimize
[DEBUG] Latest version for autoptimize: 3.1.12
[DEBUG] API download link for autoptimize: https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
[DEBUG] Stored version for autoptimize: 3.1.12
autoptimize is up-to-date and exists in mirror directory. Skipping download...
[DEBUG] Saving plugin json metadata for autoptimize version 3.1.12
[DEBUG] json metadata for autoptimize version 3.1.12 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for autoptimize.
Processing plugin: better-search-replace
[DEBUG] Checking latest version and download link for better-search-replace
[DEBUG] Latest version for better-search-replace: 1.4.7
[DEBUG] API download link for better-search-replace: https://downloads.wordpress.org/plugin/better-search-replace.zip
[DEBUG] Stored version for better-search-replace: 1.4.7
better-search-replace is up-to-date and exists in mirror directory. Skipping download...
[DEBUG] Saving plugin json metadata for better-search-replace version 1.4.7
[DEBUG] json metadata for better-search-replace version 1.4.7 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for better-search-replace.
Processing plugin: cache-enabler
[DEBUG] Checking latest version and download link for cache-enabler
[DEBUG] Latest version for cache-enabler: 1.8.15
[DEBUG] API download link for cache-enabler: https://downloads.wordpress.org/plugin/cache-enabler.1.8.15.zip
[DEBUG] Stored version for cache-enabler: 1.8.15
cache-enabler is up-to-date and exists in mirror directory. Skipping download...
[DEBUG] Saving plugin json metadata for cache-enabler version 1.8.15
[DEBUG] json metadata for cache-enabler version 1.8.15 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for cache-enabler.
Processing plugin: classic-editor
[DEBUG] Checking latest version and download link for classic-editor
[DEBUG] Latest version for classic-editor: 1.6.5
[DEBUG] API download link for classic-editor: https://downloads.wordpress.org/plugin/classic-editor.1.6.5.zip
[DEBUG] Stored version for classic-editor: 1.6.5
classic-editor is up-to-date and exists in mirror directory. Skipping download...
[DEBUG] Saving plugin json metadata for classic-editor version 1.6.5
[DEBUG] json metadata for classic-editor version 1.6.5 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for classic-editor.
Plugin download process completed.

real    0m3.881s
user    0m0.374s
sys     0m0.163s
```

Skipped download as `get_plugins_r2.sh` previously already downloaded these WordPress plugins


```bash
ls -lahrt /home/nginx/domains/plugins.domain.com/public/ 
total 6.5M
drwxr-sr-x 4 root nginx   39 Sep 30 12:56 ..
-rw-r--r-- 1 root nginx 5.9M Sep 30 18:28 advanced-custom-fields.6.3.6.zip
-rw-r--r-- 1 root nginx 101K Sep 30 18:28 akismet.5.3.3.zip
-rw-r--r-- 1 root nginx 8.9K Sep 30 18:28 amr-cron-manager.2.3.zip
-rw-r--r-- 1 root nginx  18K Sep 30 18:28 assets-manager.1.0.2.zip
-rw-r--r-- 1 root nginx 248K Sep 30 18:28 autoptimize.3.1.12.zip
-rw-r--r-- 1 root nginx 145K Sep 30 18:28 better-search-replace.1.4.7.zip
-rw-r--r-- 1 root nginx  44K Sep 30 18:29 cache-enabler.1.8.15.zip
-rw-r--r-- 1 root nginx  19K Sep 30 18:29 classic-editor.1.6.5.zip
```

Script also saves each WordPress plugin's HTTP response headers for troubleshooting

Example where source of WordPress plugin is from WordPress download on `https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip` via `x-source: WordPress`.

```bash
cat /home/nginx/domains/plugins.domain.com/plugin-logs/autoptimize.3.1.12.headers

HTTP/2 200 
date: Mon, 30 Sep 2024 18:28:57 GMT
content-type: application/zip
cf-ray: 8cb64687de35311c-LAX
cf-cache-status: MISS
access-control-allow-origin: *
cache-control: public, max-age=3600
content-disposition: attachment; filename="autoptimize.3.1.12.zip"
last-modified: Thu, 25 Jul 2024 17:15:10 GMT
access-control-allow-headers: Content-Type
access-control-allow-methods: GET, OPTIONS
x-latest-version: 3.1.12
x-nc: EXPIRED ord 8
x-source: WordPress
x-suggested-filename: autoptimize.3.1.12.zip
server: cloudflare
```

Example where source is from Cloudflare R2 S3 bucket cached version `x-source: R2`. 

Though `content-type` is JSON as the script detects that it already has the latest version of a plugin in the local mirror directory, it doesn't need to download the ZIP file again. Instead, it just updating or verifying the metadata for that plugin. The Worker is designed to handle both ZIP files and JSON metadata. When it's not necessary to download the full ZIP (because it already exists locally), the Worker returns JSON metadata about the plugin instead.

```bash
cat /home/nginx/domains/plugins.domain.com/plugin-logs/autoptimize.3.1.12.headers

HTTP/2 200 
date: Wed, 02 Oct 2024 19:42:28 GMT
content-type: application/json
content-length: 33
cf-ray: 8cc72d037ea27bcd-LAX
cf-cache-status: HIT
accept-ranges: bytes
age: 1911
last-modified: Wed, 02 Oct 2024 19:10:37 GMT
vary: Accept-Encoding
x-source: R2
server: cloudflare
```

Re-test with manual removal of the plugin

```
rm -f /home/nginx/domains/plugins.domain.com/public/autoptimize.3.1.12.zip
```

Excerpt from just `autoptimize` plugin re-download locally.

```
./get_plugins_r2.sh -p 1 -d

Processing plugin: autoptimize
[DEBUG] Checking latest version and download link for autoptimize
[DEBUG] Latest version for autoptimize: 3.1.12
[DEBUG] API download link for autoptimize: https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
[DEBUG] Stored version for autoptimize: 3.1.12
[DEBUG] API-provided download link for autoptimize: https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
[DEBUG] Constructed download link for autoptimize: https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
[DEBUG] Using API-provided download link for autoptimize: https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
[DEBUG] Downloading autoptimize version 3.1.12 through Cloudflare Worker
[DEBUG] Successfully downloaded autoptimize version 3.1.12 from R2 storage
Successfully processed autoptimize.
Time taken for autoptimize: 0.2593 seconds
[DEBUG] Saving plugin json metadata for autoptimize version 3.1.12
[DEBUG] json metadata for autoptimize version 3.1.12 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for autoptimize.

Plugin download process completed.
```

Re-inspect the saved HTTP response header for `autoptimize` with `content-type: application/zip` and `x-source: R2`.

```bash
cat /home/nginx/domains/plugins.domain.com/plugin-logs/autoptimize.3.1.12.headers

HTTP/2 200 
date: Wed, 02 Oct 2024 21:10:11 GMT
content-type: application/zip
content-length: 264379
access-control-allow-origin: *
cache-control: public, max-age=3600
content-disposition: attachment; filename="autoptimize.3.1.12.zip"
content-encoding: identity
access-control-allow-headers: Content-Type
access-control-allow-methods: GET, OPTIONS
x-source: R2
x-suggested-filename: autoptimize.3.1.12.zip
server: cloudflare
cf-ray: 8cc7ad8048d87e9c-LAX
```

Mirrored and locally cached in Cloudflare R2 bucket plugin JSON metadata where `api.mycloudflareproxy_domain.com` is Cloudflare orange cloud proxy enabled domain zone hostname enabled for Cloudflare R2 bucket public access for `WP_PLUGIN_INFO` bucket referenced in Cloudflare Worker that `get_plugins_r2.sh` talks to.

Also modified the saved JSON metadata to insert an additional field for `download_link_mirror` which also lists the mirrored download url for the WordPress plugin along with existing `download_link` download link.

```
curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/autoptimize.json | jq -r '[.download_link, .download_link_mirror]'
[
  "https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip",
  "https://downloads.mycloudflareproxy_domain.com/autoptimize.3.1.12.zip"
]
```

Full mirrored WordPress plugin JSON metadata:

```bash
curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/autoptimize.json | jq -r
```
```json
{
  "name": "Autoptimize",
  "slug": "autoptimize",
  "version": "3.1.12",
  "author": "<a href=\"https://autoptimize.com/pro/\">Frank Goossens (futtta)</a>",
  "author_profile": "https://profiles.wordpress.org/optimizingmatters/",
  "requires": "5.3",
  "tested": "6.6.2",
  "requires_php": "5.6",
  "requires_plugins": [],
  "compatibility": [],
  "rating": 94,
  "ratings": {
    "5": 1275,
    "4": 34,
    "3": 21,
    "2": 20,
    "1": 61
  },
  "num_ratings": 1411,
  "support_threads": 23,
  "support_threads_resolved": 21,
  "downloaded": 40110102,
  "last_updated": "2024-07-25 5:22pm GMT",
  "added": "2009-07-09",
  "homepage": "https://autoptimize.com/pro/",
  "sections": {
    "description": "<p>Autoptimize makes optimizing your site really easy. It can aggregate, minify and cache scripts and styles, injects CSS in the page head by default but can also inline critical CSS and defer the aggregated full CSS, moves and defers scripts to the footer and minifies HTML. You can optimize and lazy-load images (with support for WebP and AVIF formats), optimize Google Fonts, async non-aggregated JavaScript, remove WordPress core emoji cruft and more. As such it can improve your site&#8217;s performance even when already on HTTP/2! There is extensive API available to enable you to tailor Autoptimize to each and every site&#8217;s specific needs.<br />\nIf you think performance indeed is important, you should at least consider one of the many free page caching plugins (e.g. <a href=\"https://wordpress.org/plugins/speed-booster-pack/\" rel=\"ugc\">Speed Booster pack</a> or <a href=\"https://wordpress.org/plugins/cache-enabler\" rel=\"ugc\">KeyCDN&#8217;s Cache Enabler</a>) to complement Autoptimize or even <a href=\"https://misc.optimizingmatters.com/partners/?from=partnertab&amp;partner=aopro\" rel=\"nofollow ugc\">consider Autoptimize Pro</a> which not only has page caching but also image optimization, CDN, critical CSS and more!</p>\n<blockquote>\n<p><strong>Autoptimize Pro</strong><br />\n  <a href=\"https://misc.optimizingmatters.com/partners/?from=partnertab&amp;partner=aopro\" rel=\"nofollow ugc\">Autoptimize Pro is a premium Power-Up</a>, adding image optimization, CDN, page caching, automatic critical CSS rules and extra “booster” options, all in one handy subscription to <a href=\"https://misc.optimizingmatters.com/partners/?from=partnertab&amp;partner=aopro\" rel=\"nofollow ugc\">make your site even faster!</a>!</p>\n<p><strong>Premium Support</strong><br />\n  We provide great <a href=\"https://misc.optimizingmatters.com/partners/?from=partnertab&amp;partner=autoptimizepro\" rel=\"nofollow ugc\">Premium Support and Web Performance Optimization services</a> with Accelera, check out our offering on <a href=\"https://misc.optimizingmatters.com/partners/?from=partnertab&amp;partner=autoptimizepro\" rel=\"nofollow ugc\">https://accelerawp.com/</a>!</p>\n</blockquote>\n<p>(Speed-surfing image under creative commons <a href=\"https://www.flickr.com/photos/twistiti/818552808/\" rel=\"nofollow ugc\">by LL Twistiti</a>)</p>\n",
    "installation": "<p>Just install from your WordPress &#8220;Plugins &gt; Add New&#8221; screen and all will be well. Manual installation is very straightforward as well:</p>\n<ol>\n<li>Upload the zip file and unzip it in the <code>/wp-content/plugins/</code> directory</li>\n<li>Activate the plugin through the &#8216;Plugins&#8217; menu in WordPress</li>\n<li>Go to <code>Settings &gt; Autoptimize</code> and enable the options you want. Generally this means &#8220;Optimize HTML/ CSS/ JavaScript&#8221;.</li>\n</ol>\n",
    "faq": "\n<dt id='what%20does%20the%20plugin%20do%20to%20help%20speed%20up%20my%20site%3F'>\nWhat does the plugin do to help speed up my site?\n</h4>\n<p>\n<p>It minifies all scripts and styles and configures your webserver to compresses them with good expires headers. JavaScript be default will be made non-render-blocking and CSS can be too by adding critical CSS. You can configure it to combine (aggregate) CSS &amp; JS-files, in which case styles are moved to the page head, and scripts to the footer. It also minifies the HTML code and can also optimize images and Google Fonts, making your page really lightweight.</p>\n</p>\n<dt id='but%20i%27m%20on%20http%2F2%2C%20so%20i%20don%27t%20need%20autoptimize%3F'>\nBut I&#8217;m on HTTP/2, so I don&#8217;t need Autoptimize?\n</h4>\n<p>\n<p>HTTP/2 is a great step forward for sure, reducing the impact of multiple requests from the same server significantly by using the same connection to perform several concurrent requests and for that reason on new installations Autoptimize will not aggregate CSS and JS files any more. That being said, <a href=\"http://engineering.khanacademy.org/posts/js-packaging-http2.htm\" rel=\"nofollow ugc\">concatenation of CSS/ JS can still make a lot of sense</a>, as described in <a href=\"https://css-tricks.com/http2-real-world-performance-test-analysis/\" rel=\"nofollow ugc\">this css-tricks.com article</a> and this <a href=\"http://calendar.perfplanet.com/2015/packaging-for-performance/\" rel=\"nofollow ugc\">blogpost from one of the Ebay engineers</a>. The conclusion; configure, test, reconfigure, retest, tweak and look what works best in your context. Maybe it&#8217;s just HTTP/2, maybe it&#8217;s HTTP/2 + aggregation and minification, maybe it&#8217;s HTTP/2 + minification (which AO can do as well, simply untick the &#8220;aggregate JS-files&#8221; and/ or &#8220;aggregate CSS-files&#8221; options). And Autoptimize can do a lot more then &#8220;just&#8221; optimizing your JS &amp; CSS off course 😉</p>\n</p>\n<dt id='will%20this%20work%20with%20my%20blog%3F'>\nWill this work with my blog?\n</h4>\n<p>\n<p>Although Autoptimize comes without any warranties, it will in general work flawlessly if you configure it correctly. See &#8220;Troubleshooting&#8221; below for info on how to configure in case of problems. If you want you can <a href=\"https://demo.tastewp.com/autoptimize\" rel=\"nofollow ugc\">test Autoptimize on a new free dummy site, courtesy of tastewp.com</a>.</p>\n</p>\n<dt id='why%20is%20jquery.min.js%20not%20optimized%20when%20aggregating%20javascript%3F'>\nWhy is jquery.min.js not optimized when aggregating JavaScript?\n</h4>\n<p>\n<p>Starting from AO 2.1 WordPress core&#8217;s jquery.min.js is not optimized for the simple reason a lot of popular plugins inject inline JS that is not aggregated either (due to possible cache size issues with unique code in inline JS) which relies on jquery being available, so excluding jquery.min.js ensures that most sites will work out of the box. If you want optimize jquery as well, you can remove it from the JS optimization exclusion-list (you might have to enable &#8220;also aggregate inline JS&#8221; as well or switch to &#8220;force JS in head&#8221;).</p>\n</p>\n<dt id='why%20is%20autoptimized%20js%20render%20blocking%3F'>\nWhy is Autoptimized JS render blocking?\n</h4>\n<p>\n<p>This happens when aggregating JavaSCript and ticking the &#8220;force in head&#8221; option or when not aggregating and not deferring. Consider changing settings.</p>\n</p>\n<dt id='why%20is%20the%20autoptimized%20css%20still%20called%20out%20as%20render%20blocking%3F'>\nWhy is the autoptimized CSS still called out as render blocking?\n</h4>\n<p>\n<p>With the default Autoptimize configuration the CSS is linked in the head, which is a safe default but has Google PageSpeed Insights complaining. You can look into &#8220;inline all CSS&#8221; (easy) or &#8220;inline and defer CSS&#8221; (better) which are explained in this FAQ as well.</p>\n</p>\n<dt id='what%20is%20the%20use%20of%20%22inline%20and%20defer%20css%22%3F'>\nWhat is the use of &#8220;inline and defer CSS&#8221;?\n</h4>\n<p>\n<p>CSS in general should go in the head of the document. Recently a.o. Google started promoting deferring non-essential CSS, while inlining those styles needed to build the page above the fold. This is especially important to render pages as quickly as possible on mobile devices. As from Autoptimize 1.9.0 this is easy; select &#8220;inline and defer CSS&#8221;, paste the block of &#8220;above the fold CSS&#8221; in the input field (text area) and you&#8217;re good to go!</p>\n</p>\n<dt id='but%20how%20can%20one%20find%20out%20what%20the%20%22above%20the%20fold%20css%22%20is%3F'>\nBut how can one find out what the &#8220;above the fold CSS&#8221; is?\n</h4>\n<p>\n<p>There&#8217;s no easy solution for that as &#8220;above the fold&#8221; depends on where the fold is, which in turn depends on screensize. There are some tools available however, which try to identify just what is &#8220;above the fold&#8221;. <a href=\"https://github.com/addyosmani/above-the-fold-css-tools\" rel=\"nofollow ugc\">This list of tools</a> is a great starting point. The <a href=\"https://www.sitelocity.com/critical-path-css-generator\" rel=\"nofollow ugc\">Sitelocity critical CSS generator</a> and <a href=\"http://jonassebastianohlsson.com/criticalpathcssgenerator/\" rel=\"nofollow ugc\">Jonas Ohlsson&#8217;s criticalpathcssgenerator</a> are nice basic solutions and <a href=\"http://misc.optimizingmatters.com/partners/?from=faq&amp;partner=critcss\" rel=\"nofollow ugc\">http://criticalcss.com/</a> is a premium solution by the same Jonas Ohlsson. Alternatively <a href=\"https://gist.github.com/PaulKinlan/6284142\" rel=\"nofollow ugc\">this bookmarklet</a> (Chrome-only) can be helpful as well.</p>\n</p>\n<dt id='or%20should%20you%20inline%20all%20css%3F'>\nOr should you inline all CSS?\n</h4>\n<p>\n<p>The short answer: probably not. Although inlining all CSS will make the CSS non-render blocking, it will result in your base HTML-page getting significantly bigger thus requiring more &#8220;roundtrip times&#8221;. Moreover when considering multiple pages being requested in a browsing session the inline CSS is sent over each time, whereas when not inlined it would be served from cache. Finally the inlined CSS will push the meta-tags in the HTML down to a position where Facebook or Whatsapp might not look for it any more, breaking e.g. thumbnails when sharing on these platforms.</p>\n</p>\n<dt id='my%20cache%20is%20getting%20huge%2C%20doesn%27t%20autoptimize%20purge%20the%20cache%3F'>\nMy cache is getting huge, doesn&#8217;t Autoptimize purge the cache?\n</h4>\n<p>\n<p>Autoptimize does not have its proper cache purging mechanism, as this could remove optimized CSS/JS which is still referred to in other caches, which would break your site. Moreover a fast growing cache is an indication of <a href=\"http://blog.futtta.be/2016/09/15/autoptimize-cache-size-the-canary-in-the-coal-mine/\" rel=\"nofollow ugc\">other problems you should avoid</a>.</p>\n<p>Instead you can keep the cache size at an acceptable level by either:</p>\n<ul>\n<li>disactivating the &#8220;aggregate inline JS&#8221; and/ or &#8220;aggregate inline CSS&#8221; options</li>\n<li>excluding JS-variables (or sometimes CSS-selectors) that change on a per page (or per pageload) basis. You can read how you can do that <a href=\"http://blog.futtta.be/2014/03/19/how-to-keep-autoptimizes-cache-size-under-control-and-improve-visitor-experience/\" rel=\"nofollow ugc\">in this blogpost</a>.</li>\n</ul>\n<p>Despite above objections, there are 3rd party solutions to automatically purge the AO cache, e.g. using <a href=\"https://wordpress.org/support/topic/contribution-autoptimize-cache-size-under-control-by-schedule-auto-cache-purge/\" rel=\"ugc\">this code</a> or <a href=\"https://wordpress.org/plugins/bi-clean-cache/\" rel=\"ugc\">this plugin</a>, but for reasons above these are to be used only if you really know what you&#8217;re doing.</p>\n</p>\n<dt id='%22clear%20cache%22%20doesn%27t%20seem%20to%20work%3F'>\n&#8220;Clear cache&#8221; doesn&#8217;t seem to work?\n</h4>\n<p>\n<p>When clicking the &#8220;Delete Cache&#8221; link in the Autoptimize dropdown in the admin toolbar, you might to get a &#8220;Your cache might not have been purged successfully&#8221;. In that case go to Autoptimizes setting page and click the &#8220;Save changes &amp; clear cache&#8221;-button.</p>\n<p>Moreover don&#8217;t worry if your cache never is down to 0 files/ 0KB, as Autoptimize (as from version 2.2) will automatically preload the cache immediately after it has been cleared to speed further minification significantly up.</p>\n</p>\n<dt id='my%20site%20looks%20broken%20when%20i%20purge%20autoptimize%27s%20cache%21'>\nMy site looks broken when I purge Autoptimize&#8217;s cache!\n</h4>\n<p>\n<p>When clearing AO&#8217;s cache, no page cache should contain pages (HTML) that refers to the removed optimized CSS/ JS. Although for that purpose there is integration between Autoptimize and some page caches, this integration does not cover 100% of setups so you might need to purge your page cache manually.</p>\n</p>\n<dt id='can%20i%20still%20use%20cloudflare%27s%20rocket%20loader%3F'>\nCan I still use Cloudflare&#8217;s Rocket Loader?\n</h4>\n<p>\n<p>Cloudflare Rocket Loader is a pretty advanced but invasive way to make JavaScript non-render-blocking, which <a href=\"https://wordpress.org/support/topic/rocket-loader-breaking-onload-js-on-linked-css/#post-9263738\" rel=\"ugc\">Cloudflare still considers Beta</a>. Sometimes Autoptimize &amp; Rocket Loader work together, sometimes they don&#8217;t. The best approach is to disable Rocket Loader, configure Autoptimize and re-enable Rocket Loader (if you think it can help) after that and test if everything still works.</p>\n<p>At the moment (June 2017) it seems RocketLoader might break AO&#8217;s &#8220;inline &amp; defer CSS&#8221;, which is based on <a href=\"https://github.com/filamentgroup/loadCSS\" rel=\"nofollow ugc\">Filamentgroup’s loadCSS</a>, resulting in the deferred CSS not loading.</p>\n</p>\n<dt id='i%20tried%20autoptimize%20but%20my%20google%20pagespeed%20scored%20barely%20improved'>\nI tried Autoptimize but my Google Pagespeed Scored barely improved\n</h4>\n<p>\n<p>Autoptimize is not a simple &#8220;fix my Pagespeed-problems&#8221; plugin; it &#8220;only&#8221; aggregates &amp; minifies (local) JS &amp; CSS and images and allows for some nice extra&#8217;s as removing Google Fonts and deferring the loading of the CSS. As such Autoptimize will allow you to improve your performance (load time measured in seconds) and will probably also help you tackle some specific Pagespeed warnings. If you want to improve further, you will probably also have to look into e.g. page caching and your webserver configuration, which will improve real performance (again, load time as measured by e.g. https://webpagetest.org) and your &#8220;performance best practice&#8221; pagespeed ratings.</p>\n</p>\n<dt id='what%20can%20i%20do%20with%20the%20api%3F'>\nWhat can I do with the API?\n</h4>\n<p>\n<p>A whole lot; there are filters you can use to conditionally disable Autoptimize per request, to change the CSS- and JS-excludes, to change the limit for CSS background-images to be inlined in the CSS, to define what JS-files are moved behind the aggregated one, to change the defer-attribute on the aggregated JS script-tag, &#8230; There are examples for some filters in autoptimize_helper.php_example and in this FAQ.</p>\n</p>\n<dt id='how%20does%20cdn%20work%3F'>\nHow does CDN work?\n</h4>\n<p>\n<p>Starting from version 1.7.0, CDN is activated upon entering the CDN blog root directory (e.g. http://cdn.example.net/wordpress/). If that URL is present, it will used for all Autoptimize-generated files (i.e. aggregated CSS and JS), including background-images in the CSS (when not using data-uri&#8217;s).</p>\n<p>If you want your uploaded images to be on the CDN as well, you can change the upload_url_path in your WordPress configuration (/wp-admin/options.php) to the target CDN upload directory (e.g. http://cdn.example.net/wordpress/wp-content/uploads/). Do take into consideration this only works for images uploaded from that point onwards, not for images that already were uploaded. Thanks to <a href=\"https://wordpress.org/support/topic/please-don%c2%b4t-remove-cdn?replies=15#post-4720048\" rel=\"ugc\">BeautyPirate for the tip</a>!</p>\n</p>\n<dt id='why%20aren%27t%20my%20fonts%20put%20on%20the%20cdn%20as%20well%3F'>\nWhy aren&#8217;t my fonts put on the CDN as well?\n</h4>\n<p>\n<p>Autoptimize supports this, but it is not enabled by default because <a href=\"http://davidwalsh.name/cdn-fonts\" rel=\"nofollow ugc\">non-local fonts might require some extra configuration</a>. But if you have your cross-origin request policy in order, you can tell Autoptimize to put your fonts on the CDN by hooking into the API, setting <code>autoptimize_filter_css_fonts_cdn</code> to <code>true</code> this way;</p>\n<pre><code>add_filter( 'autoptimize_filter_css_fonts_cdn', '__return_true' );\n</code></pre>\n</p>\n<dt id='i%27m%20using%20cloudflare%2C%20what%20should%20i%20enter%20as%20cdn%20root%20directory'>\nI&#8217;m using Cloudflare, what should I enter as CDN root directory\n</h4>\n<p>\n<p>Nothing, when on Cloudflare your autoptimized CSS/ JS is on the Cloudflare&#8217;s CDN automatically.</p>\n</p>\n<dt id='how%20can%20i%20force%20the%20aggregated%20files%20to%20be%20static%20css%20or%20js%20instead%20of%20php%3F'>\nHow can I force the aggregated files to be static CSS or JS instead of PHP?\n</h4>\n<p>\n<p>If your webserver is properly configured to handle compression (gzip or deflate) and cache expiry (expires and cache-control with sufficient cacheability), you don&#8217;t need Autoptimize to handle that for you. In that case you can check the &#8220;Save aggregated script/css as static files?&#8221;-option, which will force Autoptimize to save the aggregated files as .css and .js-files (meaning no PHP is needed to serve these files). This setting is default as of Autoptimize 1.8.</p>\n</p>\n<dt id='how%20does%20%22exclude%20from%20optimizing%22%20work%3F'>\nHow does &#8220;exclude from optimizing&#8221; work?\n</h4>\n<p>\n<p>Both CSS and JS optimization can skip code from being aggregated and minimized by adding &#8220;identifiers&#8221; to the comma-separated exclusion list. The exact identifier string to use can be determined this way:</p>\n<ul>\n<li>if you want to exclude a specific file, e.g. wp-content/plugins/funkyplugin/css/style.css, you could simply exclude &#8220;funkyplugin/css/style.css&#8221;</li>\n<li>if you want to exclude all files of a specific plugin, e.g. wp-content/plugins/funkyplugin/js/*, you can exclude for example &#8220;funkyplugin/js/&#8221; or &#8220;plugins/funkyplugin&#8221;</li>\n<li>if you want to exclude inline code, you&#8217;ll have to find a specific, unique string in that block of code and add that to the exclusion list. Example: to exclude <code>&lt;script&gt;funky_data='Won\\'t you take me to, Funky Town'&lt;/script&gt;</code>, the identifier is &#8220;funky_data&#8221;.</li>\n</ul>\n</p>\n<dt id='troubleshooting%20autoptimize'>\nTroubleshooting Autoptimize\n</h4>\n<p>\n<p>Have a look at the troubleshooitng instructions at https://blog.futtta.be/2022/05/05/what-to-do-when-autoptimize-breaks-your-site/</p>\n</p>\n<dt id='i%20excluded%20files%20but%20they%20are%20still%20being%20autoptimized%3F'>\nI excluded files but they are still being autoptimized?\n</h4>\n<p>\n<p>AO minifies excluded JS/ CSS if the filename indicates the file is not minified yet. As of AO 2.5 you can disable this on the &#8220;JS, CSS &amp; HTML&#8221;-tab under misc. options by unticking &#8220;minify excluded files&#8221;.</p>\n</p>\n<dt id='help%2C%20i%20have%20a%20blank%20page%20or%20an%20internal%20server%20error%20after%20enabling%20autoptimize%21%21'>\nHelp, I have a blank page or an internal server error after enabling Autoptimize!!\n</h4>\n<p>\n<p>Make sure you&#8217;re not running other HTML, CSS or JS minification plugins (BWP minify, WP minify, &#8230;) simultaneously with Autoptimize or disable that functionality your page caching plugin (W3 Total Cache, WP Fastest Cache, &#8230;). Try enabling only CSS or only JS optimization to see which one causes the server error and follow the generic troubleshooting steps to find a workaround.</p>\n</p>\n<dt id='but%20i%20still%20have%20blank%20autoptimized%20css%20or%20js-files%21'>\nBut I still have blank autoptimized CSS or JS-files!\n</h4>\n<p>\n<p>If you are running Apache, the .htaccess file written by Autoptimize can in some cases conflict with the AllowOverrides settings of your Apache configuration (as is the case with the default configuration of some Ubuntu installations), which results in &#8220;internal server errors&#8221; on the autoptimize CSS- and JS-files. This can be solved by <a href=\"http://httpd.apache.org/docs/2.4/mod/core.html#allowoverride\" rel=\"nofollow ugc\">setting AllowOverrides to All</a>.</p>\n</p>\n<dt id='can%27t%20log%20in%20on%20domain%20mapped%20multisites'>\nCan&#8217;t log in on domain mapped multisites\n</h4>\n<p>\n<p>Domain mapped multisites require Autoptimize to be initialized at a different WordPress action, add this line of code to your wp-config.php to make it so to hook into <code>setup_theme</code> for example:</p>\n<pre><code>define( 'AUTOPTIMIZE_SETUP_INITHOOK', 'setup_theme' );\n</code></pre>\n</p>\n<dt id='i%20get%20no%20error%2C%20but%20my%20pages%20are%20not%20optimized%20at%20all%3F'>\nI get no error, but my pages are not optimized at all?\n</h4>\n<p>\n<p>Autoptimize does a number of checks before actually optimizing. When one of the following is true, your pages won&#8217;t be optimized:</p>\n<ul>\n<li>when in the customizer</li>\n<li>if there is no opening <code>&lt;html</code> tag</li>\n<li>if there is <code>&lt;xsl:stylesheet</code> in the response (indicating the output is not HTML but XML)</li>\n<li>if there is <code>&lt;html amp</code> in the response (as AMP-pages are optimized already)</li>\n<li>if the output is an RSS-feed (is_feed() function)</li>\n<li>if the output is a WordPress administration page (is_admin() function)</li>\n<li>if the page is requested with ?ao_noptimize=1 appended to the URL</li>\n<li>if code hooks into Autoptimize to disable optimization (see topic on Visual Composer)</li>\n<li>if other plugins use the output buffer in an incompatible manner (disable other plugins selectively to identify the culprit)</li>\n</ul>\n</p>\n<dt id='visual%20composer%2C%20beaver%20builder%20and%20similar%20page%20builder%20solutions%20are%20broken%21%21'>\nVisual Composer, Beaver Builder and similar page builder solutions are broken!!\n</h4>\n<p>\n<p>Disable the option to have Autoptimize active for logged on users and go crazy dragging and dropping 😉</p>\n</p>\n<dt id='help%2C%20my%20shop%20checkout%2F%20payment%20don%27t%20work%21%21'>\nHelp, my shop checkout/ payment don&#8217;t work!!\n</h4>\n<p>\n<p>Disable the option to optimize cart/ checkout pages (works for WooCommerce, Easy Digital Downloads and WP eCommerce).</p>\n</p>\n<dt id='revolution%20slider%20is%20broken%21'>\nRevolution Slider is broken!\n</h4>\n<p>\n<p>Make sure <code>js/jquery/jquery.min.js</code> is in the comma-separated list of JS optimization exclusions (this is excluded in the default configuration).</p>\n</p>\n<dt id='i%27m%20getting%20%22jquery%20is%20not%20defined%22%20errors'>\nI&#8217;m getting &#8220;jQuery is not defined&#8221; errors\n</h4>\n<p>\n<p>In that case you have un-aggregated JavaScript that requires jQuery to be loaded, so you&#8217;ll have to add <code>js/jquery/jquery.min.js</code> to the comma-separated list of JS optimization exclusions.</p>\n</p>\n<dt id='i%20use%20nextgen%20galleries%20and%20a%20lot%20of%20js%20is%20not%20aggregated%2F%20minified%3F'>\nI use NextGen Galleries and a lot of JS is not aggregated/ minified?\n</h4>\n<p>\n<p>NextGen Galleries does some nifty stuff to add JavaScript. In order for Autoptimize to be able to aggregate that, you can either disable Nextgen Gallery&#8217;s resourced manage with this code snippet <code>add_filter( 'run_ngg_resource_manager', '__return_false' );</code> or you can tell Autoptimize to initialize earlier, by adding this to your wp-config.php: <code>define(\"AUTOPTIMIZE_INIT_EARLIER\",\"true\");</code></p>\n</p>\n<dt id='what%20is%20noptimize%3F'>\nWhat is noptimize?\n</h4>\n<p>\n<p>Starting with version 1.6.6 Autoptimize excludes everything inside noptimize tags, e.g.:<br />\n    &lt;!&#045;&#045;noptimize&#045;&#045;&gt;&lt;script&gt;alert(&#8216;this will not get autoptimized&#8217;);&lt;/script&gt;&lt;!&#045;&#045;/noptimize&#045;&#045;&gt;</p>\n<p>You can do this in your page/ post content, in widgets and in your theme files (consider creating <a href=\"https://codex.wordpress.org/Child_Themes\" rel=\"nofollow ugc\">a child theme</a> to avoid your work being overwritten by theme updates).</p>\n</p>\n<dt id='can%20i%20change%20the%20directory%20%26%20filename%20of%20cached%20autoptimize%20files%3F'>\nCan I change the directory &amp; filename of cached autoptimize files?\n</h4>\n<p>\n<p>Yes, if you want to serve files from e.g. /wp-content/resources/aggregated_12345.css instead of the default /wp-content/cache/autoptimize/autoptimize_12345.css, then add this to wp-config.php:</p>\n<pre><code>define('AUTOPTIMIZE_CACHE_CHILD_DIR','/resources/');\ndefine('AUTOPTIMIZE_CACHEFILE_PREFIX','aggregated_');\n</code></pre>\n</p>\n<dt id='does%20this%20work%20with%20non-default%20wp_content_url%20%3F'>\nDoes this work with non-default WP_CONTENT_URL ?\n</h4>\n<p>\n<p>No, Autoptimize does not support a non-default WP_CONTENT_URL out-of-the-box, but this can be accomplished with a couple of lines of code hooking into Autoptimize&#8217;s API.</p>\n</p>\n<dt id='can%20the%20generated%20js%2F%20css%20be%20pre-gzipped%3F'>\nCan the generated JS/ CSS be pre-gzipped?\n</h4>\n<p>\n<p>Yes, but this is off by default. You can enable this by passing ´true´ to ´autoptimize_filter_cache_create_static_gzip´. You&#8217;ll obviously still have to configure your webserver to use these files instead of the non-gzipped ones to avoid the overhead of on-the-fly compression.</p>\n</p>\n<dt id='what%20does%20%22remove%20emojis%22%20do%3F'>\nWhat does &#8220;remove emojis&#8221; do?\n</h4>\n<p>\n<p>This new option in Autoptimize 2.3 removes the inline CSS, inline JS and linked JS-file added by WordPress core. As such is can have a small positive impact on your site&#8217;s performance.</p>\n</p>\n<dt id='is%20%22remove%20query%20strings%22%20useful%3F'>\nIs &#8220;remove query strings&#8221; useful?\n</h4>\n<p>\n<p>Although some online performance assessment tools will single out &#8220;query strings for static files&#8221; as an issue for performance, in general the impact of these is almost non-existant. As such Autoptimize, since version 2.3, allows you to have the query string (or more precisely the &#8220;ver&#8221;-parameter) removed, but ticking &#8220;remove query strings from static resources&#8221; will have little or no impact of on your site&#8217;s performance as measured in (milli-)seconds.</p>\n</p>\n<dt id='%28how%29%20should%20i%20optimize%20google%20fonts%3F'>\n(How) should I optimize Google Fonts?\n</h4>\n<p>\n<p>Google Fonts are typically loaded by a &#8220;render blocking&#8221; linked CSS-file. If you have a theme and plugins that use Google Fonts, you might end up with multiple such CSS-files. Autoptimize (since version 2.3) now let&#8217;s you lessen the impact of Google Fonts by either removing them alltogether or by optimizing the way they are loaded. There are two optimization-flavors; the first one is &#8220;combine and link&#8221;, which replaces all requests for Google Fonts into one request, which will still be render-blocking but will allow the fonts to be loaded immediately (meaning you won&#8217;t see fonts change while the page is loading). The alternative is &#8220;combine and load async&#8221; which uses JavaScript to load the fonts in a non-render blocking manner but which might cause a &#8220;flash of unstyled text&#8221;.</p>\n</p>\n<dt id='should%20i%20use%20%22preconnect%22'>\nShould I use &#8220;preconnect&#8221;\n</h4>\n<p>\n<p>Preconnect is a somewhat advanced feature to instruct browsers (<a href=\"https://caniuse.com/#feat=link-rel-preconnect\" rel=\"nofollow ugc\">if they support it</a>) to make a connection to specific domains even if the connection is not immediately needed. This can be used e.g. to lessen the impact of 3rd party resources on HTTPS (as DNS-request, TCP-connection and SSL/TLS negotiation are executed early). Use with care, as preconnecting to too many domains can be counter-productive.</p>\n</p>\n<dt id='when%20can%28%27t%29%20i%20async%20js%3F'>\nWhen can(&#8216;t) I async JS?\n</h4>\n<p>\n<p>JavaScript files that are not autoptimized (because they were excluded or because they are hosted elsewhere) are typically render-blocking. By adding them in the comma-separated &#8220;async JS&#8221; field, Autoptimize will add the async flag causing the browser to load those files asynchronously (i.e. non-render blocking). This can however break your site (page), e.g. if you async &#8220;js/jquery/jquery.min.js&#8221; you will very likely get &#8220;jQuery is not defined&#8221;-errors. Use with care.</p>\n</p>\n<dt id='how%20does%20image%20optimization%20work%3F'>\nHow does image optimization work?\n</h4>\n<p>\n<p>When image optimization is on, Autoptimize will look for png, gif, jpeg (.jpg) files in image tags and in your CSS files that are loaded from your own domain and change the src (source) to the ShortPixel CDN for those. Important: this can only work for publicly available images, otherwise the image optimization proxy will not be able to get the image to optimize it, so firewalls or proxies or password protection or even hotlinking-prevention might break image optimization.</p>\n</p>\n<dt id='can%20i%20use%20image%20optimization%20for%20my%20intranet%2F%20protected%20site%3F'>\nCan I use image optimization for my intranet/ protected site?\n</h4>\n<p>\n<p>No; Image optimization depends on the ability of the external image optimization service to fetch the original image from your site, optimize it and save it on the CDN. If you images cannot be downloaded by anonymous visitors (due to firewall/ proxy/ password protection/ hotlinking-protection), image optimization will not work.</p>\n</p>\n<dt id='where%20can%20i%20get%20more%20info%20on%20image%20optimization%3F'>\nWhere can I get more info on image optimization?\n</h4>\n<p>\n<p>Have a look at <a href=\"https://shortpixel.helpscoutdocs.com/category/60-shortpixel-ai-cdn\" rel=\"nofollow ugc\">Shortpixel&#8217;s FAQ</a>.</p>\n</p>\n<dt id='can%20i%20disable%20ao%20listening%20to%20page%20cache%20purges%3F'>\nCan I disable AO listening to page cache purges?\n</h4>\n<p>\n<p>As from AO 2.4 AO &#8220;listens&#8221; to page cache purges to clear its own cache. You can disable this behavior with this filter;</p>\n<pre><code>add_filter('autoptimize_filter_main_hookpagecachepurge','__return_false');\n</code></pre>\n</p>\n<dt id='some%20of%20the%20non-ascii%20characters%20get%20lost%20after%20optimization'>\nSome of the non-ASCII characters get lost after optimization\n</h4>\n<p>\n<p>By default AO uses non multibyte-safe string methods, but if your PHP has the mbstring extension you can enable multibyte-safe string functions with this filter;</p>\n<pre><code>add_filter('autoptimize_filter_main_use_mbstring', '__return_true');\n</code></pre>\n</p>\n<dt id='i%20can%27t%20get%20critical%20css%20working'>\nI can&#8217;t get Critical CSS working\n</h4>\n<p>\n<p>Check <a href=\"https://wordpress.org/plugins/autoptimize-criticalcss/#faq\" rel=\"ugc\">the FAQ on the (legacy) &#8220;power-up&#8221; here</a>, this info will be integrated in this FAQ at a later date.</p>\n</p>\n<dt id='do%20i%20still%20need%20the%20critical%20css%20power-up%20when%20i%20have%20autoptimize%202.7%20or%20higher%3F'>\nDo I still need the Critical CSS power-up when I have Autoptimize 2.7 or higher?\n</h4>\n<p>\n<p>No, the Critical CSS power-up is not needed any more, all functionality (and many fixes/ improvements) are now part of Autoptimize.</p>\n</p>\n<dt id='what%20does%20%22enable%20404%20fallbacks%22%20do%3F%20why%20would%20i%20need%20this%3F'>\nWhat does &#8220;enable 404 fallbacks&#8221; do? Why would I need this?\n</h4>\n<p>\n<p>Autoptimize caches aggregated &amp; optimized CSS/ JS and links to those cached files are stored in the HTML, which will be stored in a page cache (which can be a plugin, can be at host level, can be at 3rd party, in the Google cache, in a browser). If there is HTML in a page cache that links to Autoptimized CSS/ JS that has been removed in the mean time (when the cache was cleared) then the page from cache will not look/ work as expected as the CSS or JS were not found (a 404 error).</p>\n<p>This setting aims to prevent things from breaking by serving &#8220;fallback&#8221; CSS or JS. The fallback-files are copies of the first Autoptimized CSS &amp; JS files created after the cache was emptied and as such will based on the homepage. This means that the CSS/ JS migth not apply 100% on other pages, but at least the impact of missing CSS/ JS will be lessened (often significantly).</p>\n<p>When the option is enabled, Autoptimize adds an <code>ErrorDocument 404</code> to the .htaccess (as used by Apache) and will also hook into WordPress core <code>template_redirect</code> to capture 404&#8217;s handled by WordPress. When using NGINX something like below should work (I&#8217;m not an NGINX specialist, but it does work for me);</p>\n<pre><code>location ~* /wp-content/cache/autoptimize/.*\\.(js|css)$ {\n    try_files $uri $uri/ /wp-content/autoptimize_404_handler.php;\n}\n</code></pre>\n<p>And this a nice alternative approach (provided by fboylovesyou);</p>\n<pre><code>location ~* /wp-content/cache/autoptimize/.*\\.(css)$ {\n    try_files $uri $uri/ /wp-content/cache/autoptimize/css/autoptimize_fallback.css;\n}\nlocation ~* /wp-content/cache/autoptimize/.*\\.(js)$ {\n    try_files $uri $uri/ /wp-content/cache/autoptimize/js/autoptimize_fallback.js;\n}\n</code></pre>\n</p>\n<dt id='what%20open%20source%20software%2F%20projects%20are%20used%20in%20autoptimize%3F'>\nWhat open source software/ projects are used in Autoptimize?\n</h4>\n<p>\n<p>The following great open source projects are used in Autoptimize in some form or another:</p>\n<ul>\n<li><a href=\"https://github.com/mrclay/minify/\" rel=\"nofollow ugc\">Mr Clay&#8217;s Minify</a> for JS &amp; HTML minification</li>\n<li><a href=\"https://github.com/tubalmartin/YUI-CSS-compressor-PHP-port\" rel=\"nofollow ugc\">YUI CSS compressor PHP Port</a> for CSS minification</li>\n<li><a href=\"https://github.com/aFarkas/lazysizes\" rel=\"nofollow ugc\">Lazysizes</a> for lazyload</li>\n<li><a href=\"https://github.com/w3guy/persist-admin-notices-dismissal\" rel=\"nofollow ugc\">Persist Admin Notices Dismissal</a> for notices in the administration screens</li>\n<li><a href=\"https://github.com/YahnisElsts/plugin-update-checker/\" rel=\"nofollow ugc\">Plugin Update Checker</a> for automated updates from Github for the beta version</li>\n<li><a href=\"https://github.com/filamentgroup/loadCSS\" rel=\"nofollow ugc\">LoadCSS</a> for deferring full CSS</li>\n<li><a href=\"https://github.com/carhartl/jquery-cookie\" rel=\"nofollow ugc\">jQuery cookie</a> to store the &#8220;futtta about&#8221; category selection in a cookie</li>\n<li><a href=\"https://github.com/christianbach/tablesorter\" rel=\"nofollow ugc\">jQuery tablesorter</a> for the critical CSS rules/ jobs display</li>\n<li><a href=\"https://github.com/idiot/unslider/\" rel=\"nofollow ugc\">jQuery unslider</a> for the mini-slider in the top right corner on the main settings page (repo gone)</li>\n<li><a href=\"https://github.com/blueimp/JavaScript-MD5\" rel=\"nofollow ugc\">JavaScript-md5</a> for critical CSS rules editing</li>\n<li><a href=\"https://wordpress.org/plugins/speed-booster-pack/\" rel=\"ugc\">Speed Booster Pack</a> for advanced JS deferring</li>\n<li><a href=\"https://wordpress.org/plugins/disable-remove-google-fonts/\" rel=\"ugc\">Disable Remove Google Fonts</a> for additional Google Font removal</li>\n</ul>\n</p>\n<dt id='where%20can%20i%20get%20help%3F'>\nWhere can I get help?\n</h4>\n<p>\n<p>You can get help on the <a href=\"https://wordpress.org/support/plugin/autoptimize\" rel=\"ugc\">wordpress.org support forum</a>. If you are 100% sure this your problem cannot be solved using Autoptimize configuration and that you in fact discovered a bug in the code, you can <a href=\"https://github.com/futtta/autoptimize/issues\" rel=\"nofollow ugc\">create an issue on GitHub</a>. If you&#8217;re looking for premium support, check out our <a href=\"http://autoptimize.com/\" rel=\"nofollow ugc\">Autoptimize Pro Support and Web Performance Optimization services</a>.</p>\n</p>\n<dt id='i%20want%20out%2C%20how%20should%20i%20remove%20autoptimize%3F'>\nI want out, how should I remove Autoptimize?\n</h4>\n<p>\n<ul>\n<li>Disable the plugin (this will remove options and cache)</li>\n<li>Remove the plugin</li>\n<li>Clear any cache that might still have pages which reference Autoptimized CSS/JS (e.g. of a page caching plugin such as WP Super Cache)</li>\n</ul>\n</p>\n<dt id='how%20can%20i%20help%2F%20contribute%3F'>\nHow can I help/ contribute?\n</h4>\n<p>\n<p>Just <a href=\"https://github.com/futtta/autoptimize\" rel=\"nofollow ugc\">fork Autoptimize on Github</a> and code away!</p>\n</p>\n\n",
    "changelog": "<h4>3.1.12</h4>\n<ul>\n<li>image optimization: improvements to the favicon regex</li>\n<li>javascript optimization: integrate most recent version of jsmin.php</li>\n<li>critical CSS: improve blocklist (url/ paths that should not be added to the job queue)</li>\n<li>some other minor changes/ improvements/ filters, see the <a href=\"https://github.com/futtta/autoptimize/commits/beta\" rel=\"nofollow ugc\">GitHub commit log</a>.</li>\n</ul>\n<h4>3.1.11</h4>\n<ul>\n<li>code quality improvements see the <a href=\"https://github.com/futtta/autoptimize/commits/beta\" rel=\"nofollow ugc\">GitHub commit log</a>.</li>\n<li>some other minor changes/ improvements/ filters, see the <a href=\"https://github.com/futtta/autoptimize/commits/beta\" rel=\"nofollow ugc\">GitHub commit log</a>.</li>\n</ul>\n<h4>3.1.10</h4>\n<ul>\n<li>improvement: with &#8220;don&#8217;t aggregate but defer&#8221; and &#8220;also defer inline JS&#8221; on, also defer JS that had the async flag to avoid the (previously) asynced JS from executing before the inline JS has ran.</li>\n<li>improvement: show option to disable the default on &#8220;compatibility logic&#8221;.</li>\n<li>fix for regression in  3.1.9 which caused JetPack Image optimization not working even if image optimization was off in AO.</li>\n<li>API: some extra hooks in critical CSS to enable others (and AOPro) to act on changes in critical CSS rules</li>\n<li>some other minor changes/ improvements/ filters, see the <a href=\"https://github.com/futtta/autoptimize/commits/beta\" rel=\"nofollow ugc\">GitHub commit log</a>.</li>\n</ul>\n<h4>3.1.9</h4>\n<ul>\n<li>improvement: activate JS, CSS &amp; HTML optimization upon plugin activation (hat tip to Adam Silverstein (developer relations engineer at Google))</li>\n<li>improvement: also defer asynced JS (to ensure execution order remains intact; asynced JS should not execute before deferred inline JS which it might depend upon)</li>\n<li>improvement: exclude images from being lazyloaded if they have fetchpriority attribute set to high (as done by WordPress core since 6.3)</li>\n<li>bugfix: disable spellcheck on CSS textarea&#8217;s (above the fold CSS/ critical CSS) which in some cases caused browser issues</li>\n<li>add tab to explain Autoptimize Pro.</li>\n<li>confirmed working with WordPress 6.4 (beta 3)</li>\n<li>some other minor changes/ improvements/ filters, see the <a href=\"https://github.com/futtta/autoptimize/commits/beta\" rel=\"nofollow ugc\">GitHub commit log</a>.</li>\n</ul>\n<h4>3.1.8.1</h4>\n<ul>\n<li>urgent fix for PHP error, sorry about that!</li>\n</ul>\n<h4>3.1.8</h4>\n<ul>\n<li>Images: improve optmization logic for background images</li>\n<li>Critical CSS: don&#8217;t trigger custom_post rule if not is_singular + adding debug logging for rule selection</li>\n<li>some other minor changes/ improvements/ filters, see the <a href=\"https://github.com/futtta/autoptimize/commits/beta\" rel=\"nofollow ugc\">GitHub commit log</a>.</li>\n</ul>\n<h4>3.1.7</h4>\n<ul>\n<li>security: improve validation (import) and sanitization (output) of critical CSS rules, to fix a medium severity Admin+ Stored Cross-Site Scripting vulnerability as reported by WP Scan Security.</li>\n</ul>\n<h4>3.1.6</h4>\n<ul>\n<li>CSS: removing trailing slashes in &lt;link tags for more W3 HTML validation love</li>\n<li>Extra: also dequeue WooCommerce block CSS if &#8220;remove WordPress block CSS&#8221; option is active</li>\n<li>imgopt: also act on non-aggregated inline CSS</li>\n<li>imgopt: added logic to warn users if Shortpixel can&#8217;t reach their site</li>\n<li>backend: AO toolbar JS/ CSS is finally minified as well.</li>\n<li>explicitly disable optimization of login pages</li>\n<li>some other minor changes/ improvements/ filters, see the <a href=\"https://github.com/futtta/autoptimize/commits/beta\" rel=\"nofollow ugc\">GitHub commit log</a>.</li>\n</ul>\n<p><h4>3.1.5</h4>\n</p>\n<ul>\n<li>improvements to JSMin by Robert Ehrenleitner (big thanks Robert!).</li>\n<li>do not consider jquery.js as minified any more (WordPress now uses jquery.min.js by default and jquery.js is the unminified version).</li>\n<li>fix for &#8220;undefined array key&#8221; PHP errors in autoptimizeCriticalCSSCron.php</li>\n<li>some other minor changes/ improvements/ filters, see the <a href=\"https://github.com/futtta/autoptimize/commits/beta\" rel=\"nofollow ugc\">GitHub commit log</a>.</li>\n</ul>\n<p><h4>3.1.4</h4>\n</p>\n<ul>\n<li>Improvement: when all CSS is inlined, try doing so after SEO meta-tags (just before ld+json script tag which most SEO plugins add as last item on their list).</li>\n<li>Img opt: also optimize images set in data-background and data-retina attributes (+ filter to easily add other attributes)</li>\n<li>CSS opt: filter to enable AO to skip minification of calc formulas in CSS (as the CSS minifier on rare occasions breaks those)</li>\n<li>Multiple other filters added</li>\n<li>Some other minor changes/ improvements/ filters, see the <a href=\"https://github.com/futtta/autoptimize/commits/beta\" rel=\"nofollow ugc\">GitHub commit log</a>.</li>\n</ul>\n<p><h4>3.1.3</h4>\n</p>\n<ul>\n<li>Multiple fixes for metabox LCP image preloads (thanks <a href=\"https://foxscribbler.com/\" rel=\"nofollow ugc\">Kishorchand</a> for notifying &amp; providing a staging environment to debug on).</li>\n<li>Fix in revslider compatibility (hat tip <a href=\"https://wordpress.org/support/topic/issue-with-latest-version-of-slider-revolution/\" rel=\"ugc\">Waqar Ahmed for reporting &amp; helping out</a> ).</li>\n<li>No image optimization or criticalcss attempts on localhost installations any more + notification of that fact if localhost detected.</li>\n<li>Some other minor changes/ improvements/ filters, see the <a href=\"https://github.com/futtta/autoptimize/commits/beta\" rel=\"nofollow ugc\">GitHub commit log</a>.</li>\n</ul>\n<p><h4>3.1.2</h4>\n</p>\n<ul>\n<li>Google Fonts: some more removal logic</li>\n<li>fix for 404 fallback bug (hat tip to Asif for finding &amp; reporting)</li>\n<li>Some other minor changes/ improvements/ filters, see the <a href=\"https://github.com/futtta/autoptimize/commits/beta\" rel=\"nofollow ugc\">GitHub commit log</a>.</li>\n</ul>\n<p><h4>3.1.1.1</h4>\n</p>\n<ul>\n<li>Quick workaround for an autoload conflict with JetFormBuilder (and maybe other Crocoblock plugins?) that causes a critical error on the AO settings page.</li>\n</ul>\n<p><h4>3.1.1</h4>\n</p>\n<ul>\n<li>images: when optimizing images and lazyloading is on, then by default do not set an LQIP (low quality image placeholder) any more (reason: it might <em>look</em> nice but it comes with a small-ish perf. penalty). This can be re-enabled by returning true to the <code>autoptimize_filter_imgopt_lazyload_dolqip</code> filter.</li>\n<li>security: further improvements to critical CSS settings page (again with the great assistance of WPScan Security).</li>\n<li>some other minor changes/ improvements/ filters, see the <a href=\"https://github.com/futtta/autoptimize/commits/beta\" rel=\"nofollow ugc\">GitHub commit log</a>.</li>\n</ul>\n<p><h4>3.1.0</h4>\n</p>\n<ul>\n<li>new HTML sub-option: &#8220;minify inline CSS/ JS&#8221; (off by default).</li>\n<li>new Misc option: permanently allow the &#8220;do not run compatibility logic&#8221; flag to be removed (which was set for users upgrading from AO 2.9.* to AO 3.0.* as the assumption was things were working anyway).</li>\n<li>security: improvements to the critical CSS settings page to fix authenticated cross site scripting issues as reported by WPScan Security.</li>\n<li>bugfix: &#8220;defer inline JS&#8221; of very large chunks of inline JS could cause server errors (PCRE crash actually) so not deferring if string is more then 200000 characters (filter available).</li>\n<li>some other minor changes/ improvements/ hooks, see the <a href=\"https://github.com/futtta/autoptimize/commits/beta\" rel=\"nofollow ugc\">GitHub commit log</a></li>\n</ul>\n<p><h4>3.0.4</h4>\n</p>\n<ul>\n<li>fix for &#8220;undefined array key ao_post_preload” on post/ page edit screens</li>\n<li>fix for image optimization altering inline JS that contains an <code>&lt;img</code> tag if lazyload is not active</li>\n<li>improvements to exit survey</li>\n<li>confirmed working with WordPress 6.0</li>\n</ul>\n<p><h4>3.0.3</h4>\n</p>\n<ul>\n<li>fix for images being preloaded without this being configured when lazyload is on and per page/post settings are off.</li>\n<li>ensure critical CSS schedule is always known.</li>\n<li>when deferring non-aggregated JS, make the optimatization exclusions take the full script-tag into account instead of just the src URL.</li>\n</ul>\n<p><h4>3.0.2</h4>\n</p>\n<ul>\n<li>rollback automatic &#8220;minify inline CSS/ JS&#8221; which broke more then expected, this will come back as a separate default off option later and can now be enabled with a simple filter: <code>add_filter( 'autoptimize_html_minify_inline_js_css', '__return_true');</code> .</li>\n<li>fix for &#8220;Call to undefined method autoptimizeOptionWrapper::delete_option()&#8221; in autoptimizeVersionUpdatesHandler.php</li>\n</ul>\n<p><h4>3.0.1</h4>\n</p>\n<ul>\n<li>fix for minification of inline script with type text/template breaking the template (e.g. ninja forms), hat tip to @bobsled.</li>\n<li>fix for regression in import of CSS-files where e.g. fontawesome CSS was broken due to being escaped again with help of @bobsled, thanks man!</li>\n</ul>\n<p><h4>3.0.0</h4>\n</p>\n<ul>\n<li>fundamental change for new installations: by default Autoptimize will not aggregate JS/ CSS any more (HTTP/2 is ubiquitous and there are other advantages to not aggregating esp. re. inline JS/ CSS and dependancies)</li>\n<li>new: no API needed any more to create manual critical CSS rules. </li>\n<li>new: &#8220;Remove WordPress blocks CSS&#8221; option on the &#8220;Extra&#8221; tab to remove block- and global styles (and SVG).</li>\n<li>new: compatibility logic for &#8220;edit with elementor&#8221;, &#8220;revolution slider&#8221;, for non-aggregated inline JS requiring jQuery even if not excluded (= auto-exclude of jQuery) and JS-heavy WordPress blocks (Gutenberg)</li>\n<li>new: configure an image to be preloaded on a per page/ post basis for better LCP.</li>\n<li>improvement: defer inline now also allowed if inline JS contains nonce or post_id.</li>\n<li>improvement: settings export/ import on critical CSS tab now takes into account all Autoptimize settings, not just the critical CSS ones.</li>\n<li>technical improvement: all criticalCSS classes were refactored, removing use of global variables.</li>\n<li>technical improvement: automated unit tests on Travis-CI for PHP versions 7.2 to 8.1.</li>\n<li>fix: stop Divi from clearing Autoptimize&#8217;s cache <a href=\"https://blog.futtta.be/2018/11/17/warning-divi-purging-autoptimizes-cache/\" rel=\"nofollow ugc\">which is pretty counter-productive</a>.</li>\n<li>misc smaller fixes/ improvements, see the <a href=\"https://github.com/futtta/autoptimize/commits/beta\" rel=\"nofollow ugc\">GitHub commit log</a></li>\n</ul>\n<p><h4>older</h4>\n</p>\n<ul>\n<li>see <a href=\"https://plugins.svn.wordpress.org/autoptimize/tags/2.9.5.1/readme.txt\" rel=\"nofollow ugc\">https://plugins.svn.wordpress.org/autoptimize/tags/2.9.5.1/readme.txt</a></li>\n</ul>\n"
  },
  "download_link": "https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip",
  "download_link_mirror": "https://downloads.mycloudflareproxy_domain.com/autoptimize.3.1.12.zip",
  "screenshots": [],
  "tags": {
    "core-web-vitals": "core web vitals",
    "images": "images",
    "optimize": "Optimize",
    "pagespeed": "pagespeed",
    "performance": "performance"
  },
  "versions": {
    "0.1": "https://downloads.wordpress.org/plugin/autoptimize.0.1.zip",
    "0.2": "https://downloads.wordpress.org/plugin/autoptimize.0.2.zip",
    "0.3": "https://downloads.wordpress.org/plugin/autoptimize.0.3.zip",
    "0.4": "https://downloads.wordpress.org/plugin/autoptimize.0.4.zip",
    "0.5": "https://downloads.wordpress.org/plugin/autoptimize.0.5.zip",
    "0.6": "https://downloads.wordpress.org/plugin/autoptimize.0.6.zip",
    "0.7": "https://downloads.wordpress.org/plugin/autoptimize.0.7.zip",
    "0.8": "https://downloads.wordpress.org/plugin/autoptimize.0.8.zip",
    "0.9": "https://downloads.wordpress.org/plugin/autoptimize.0.9.zip",
    "1.1": "https://downloads.wordpress.org/plugin/autoptimize.1.1.zip",
    "1.2": "https://downloads.wordpress.org/plugin/autoptimize.1.2.zip",
    "1.3": "https://downloads.wordpress.org/plugin/autoptimize.1.3.zip",
    "1.4": "https://downloads.wordpress.org/plugin/autoptimize.1.4.zip",
    "1.5": "https://downloads.wordpress.org/plugin/autoptimize.1.5.zip",
    "1.5.1": "https://downloads.wordpress.org/plugin/autoptimize.1.5.1.zip",
    "1.6.0": "https://downloads.wordpress.org/plugin/autoptimize.1.6.0.zip",
    "1.6.1": "https://downloads.wordpress.org/plugin/autoptimize.1.6.1.zip",
    "1.6.2": "https://downloads.wordpress.org/plugin/autoptimize.1.6.2.zip",
    "1.6.3": "https://downloads.wordpress.org/plugin/autoptimize.1.6.3.zip",
    "1.6.4": "https://downloads.wordpress.org/plugin/autoptimize.1.6.4.zip",
    "1.6.5": "https://downloads.wordpress.org/plugin/autoptimize.1.6.5.zip",
    "1.6.6": "https://downloads.wordpress.org/plugin/autoptimize.1.6.6.zip",
    "1.7.0": "https://downloads.wordpress.org/plugin/autoptimize.1.7.0.zip",
    "1.7.1": "https://downloads.wordpress.org/plugin/autoptimize.1.7.1.zip",
    "1.7.2": "https://downloads.wordpress.org/plugin/autoptimize.1.7.2.zip",
    "1.7.3": "https://downloads.wordpress.org/plugin/autoptimize.1.7.3.zip",
    "1.8.0": "https://downloads.wordpress.org/plugin/autoptimize.1.8.0.zip",
    "1.8.1": "https://downloads.wordpress.org/plugin/autoptimize.1.8.1.zip",
    "1.8.2": "https://downloads.wordpress.org/plugin/autoptimize.1.8.2.zip",
    "1.8.3": "https://downloads.wordpress.org/plugin/autoptimize.1.8.3.zip",
    "1.8.4": "https://downloads.wordpress.org/plugin/autoptimize.1.8.4.zip",
    "1.8.5": "https://downloads.wordpress.org/plugin/autoptimize.1.8.5.zip",
    "1.9.0": "https://downloads.wordpress.org/plugin/autoptimize.1.9.0.zip",
    "1.9.1": "https://downloads.wordpress.org/plugin/autoptimize.1.9.1.zip",
    "1.9.2": "https://downloads.wordpress.org/plugin/autoptimize.1.9.2.zip",
    "1.9.3": "https://downloads.wordpress.org/plugin/autoptimize.1.9.3.zip",
    "1.9.4": "https://downloads.wordpress.org/plugin/autoptimize.1.9.4.zip",
    "2.0.0": "https://downloads.wordpress.org/plugin/autoptimize.2.0.0.zip",
    "2.0.1": "https://downloads.wordpress.org/plugin/autoptimize.2.0.1.zip",
    "2.0.2": "https://downloads.wordpress.org/plugin/autoptimize.2.0.2.zip",
    "2.1.0": "https://downloads.wordpress.org/plugin/autoptimize.2.1.0.zip",
    "2.1.1": "https://downloads.wordpress.org/plugin/autoptimize.2.1.1.zip",
    "2.1.2": "https://downloads.wordpress.org/plugin/autoptimize.2.1.2.zip",
    "2.2.0": "https://downloads.wordpress.org/plugin/autoptimize.2.2.0.zip",
    "2.2.1": "https://downloads.wordpress.org/plugin/autoptimize.2.2.1.zip",
    "2.2.2": "https://downloads.wordpress.org/plugin/autoptimize.2.2.2.zip",
    "2.3.0": "https://downloads.wordpress.org/plugin/autoptimize.2.3.0.zip",
    "2.3.1": "https://downloads.wordpress.org/plugin/autoptimize.2.3.1.zip",
    "2.3.2": "https://downloads.wordpress.org/plugin/autoptimize.2.3.2.zip",
    "2.3.3": "https://downloads.wordpress.org/plugin/autoptimize.2.3.3.zip",
    "2.3.4": "https://downloads.wordpress.org/plugin/autoptimize.2.3.4.zip",
    "2.4.0": "https://downloads.wordpress.org/plugin/autoptimize.2.4.0.zip",
    "2.4.1": "https://downloads.wordpress.org/plugin/autoptimize.2.4.1.zip",
    "2.4.2": "https://downloads.wordpress.org/plugin/autoptimize.2.4.2.zip",
    "2.4.3": "https://downloads.wordpress.org/plugin/autoptimize.2.4.3.zip",
    "2.4.4": "https://downloads.wordpress.org/plugin/autoptimize.2.4.4.zip",
    "2.5.0": "https://downloads.wordpress.org/plugin/autoptimize.2.5.0.zip",
    "2.5.1": "https://downloads.wordpress.org/plugin/autoptimize.2.5.1.zip",
    "2.6.0": "https://downloads.wordpress.org/plugin/autoptimize.2.6.0.zip",
    "2.6.1": "https://downloads.wordpress.org/plugin/autoptimize.2.6.1.zip",
    "2.6.2": "https://downloads.wordpress.org/plugin/autoptimize.2.6.2.zip",
    "2.7.0": "https://downloads.wordpress.org/plugin/autoptimize.2.7.0.zip",
    "2.7.1": "https://downloads.wordpress.org/plugin/autoptimize.2.7.1.zip",
    "2.7.2": "https://downloads.wordpress.org/plugin/autoptimize.2.7.2.zip",
    "2.7.3": "https://downloads.wordpress.org/plugin/autoptimize.2.7.3.zip",
    "2.7.4": "https://downloads.wordpress.org/plugin/autoptimize.2.7.4.zip",
    "2.7.5": "https://downloads.wordpress.org/plugin/autoptimize.2.7.5.zip",
    "2.7.6": "https://downloads.wordpress.org/plugin/autoptimize.2.7.6.zip",
    "2.7.7": "https://downloads.wordpress.org/plugin/autoptimize.2.7.7.zip",
    "2.7.8": "https://downloads.wordpress.org/plugin/autoptimize.2.7.8.zip",
    "2.8.0": "https://downloads.wordpress.org/plugin/autoptimize.2.8.0.zip",
    "2.8.1": "https://downloads.wordpress.org/plugin/autoptimize.2.8.1.zip",
    "2.8.2": "https://downloads.wordpress.org/plugin/autoptimize.2.8.2.zip",
    "2.8.3": "https://downloads.wordpress.org/plugin/autoptimize.2.8.3.zip",
    "2.8.4": "https://downloads.wordpress.org/plugin/autoptimize.2.8.4.zip",
    "2.9.0": "https://downloads.wordpress.org/plugin/autoptimize.2.9.0.zip",
    "2.9.1": "https://downloads.wordpress.org/plugin/autoptimize.2.9.1.zip",
    "2.9.2": "https://downloads.wordpress.org/plugin/autoptimize.2.9.2.zip",
    "2.9.3": "https://downloads.wordpress.org/plugin/autoptimize.2.9.3.zip",
    "2.9.4": "https://downloads.wordpress.org/plugin/autoptimize.2.9.4.zip",
    "2.9.5": "https://downloads.wordpress.org/plugin/autoptimize.2.9.5.zip",
    "2.9.5.1": "https://downloads.wordpress.org/plugin/autoptimize.2.9.5.1.zip",
    "3.0.0": "https://downloads.wordpress.org/plugin/autoptimize.3.0.0.zip",
    "3.0.1": "https://downloads.wordpress.org/plugin/autoptimize.3.0.1.zip",
    "3.0.2": "https://downloads.wordpress.org/plugin/autoptimize.3.0.2.zip",
    "3.0.3": "https://downloads.wordpress.org/plugin/autoptimize.3.0.3.zip",
    "3.0.4": "https://downloads.wordpress.org/plugin/autoptimize.3.0.4.zip",
    "3.1.0": "https://downloads.wordpress.org/plugin/autoptimize.3.1.0.zip",
    "3.1.1": "https://downloads.wordpress.org/plugin/autoptimize.3.1.1.zip",
    "3.1.1.1": "https://downloads.wordpress.org/plugin/autoptimize.3.1.1.1.zip",
    "3.1.10": "https://downloads.wordpress.org/plugin/autoptimize.3.1.10.zip",
    "3.1.11": "https://downloads.wordpress.org/plugin/autoptimize.3.1.11.zip",
    "3.1.12": "https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip",
    "3.1.2": "https://downloads.wordpress.org/plugin/autoptimize.3.1.2.zip",
    "3.1.3": "https://downloads.wordpress.org/plugin/autoptimize.3.1.3.zip",
    "3.1.4": "https://downloads.wordpress.org/plugin/autoptimize.3.1.4.zip",
    "3.1.5": "https://downloads.wordpress.org/plugin/autoptimize.3.1.5.zip",
    "3.1.6": "https://downloads.wordpress.org/plugin/autoptimize.3.1.6.zip",
    "3.1.7": "https://downloads.wordpress.org/plugin/autoptimize.3.1.7.zip",
    "3.1.8": "https://downloads.wordpress.org/plugin/autoptimize.3.1.8.zip",
    "3.1.8.1": "https://downloads.wordpress.org/plugin/autoptimize.3.1.8.1.zip",
    "3.1.9": "https://downloads.wordpress.org/plugin/autoptimize.3.1.9.zip",
    "trunk": "https://downloads.wordpress.org/plugin/autoptimize.zip"
  },
  "donate_link": "http://blog.futtta.be/2013/10/21/do-not-donate-to-me/",
  "contributors": {
    "futtta": "https://profiles.wordpress.org/futtta/",
    "optimizingmatters": "https://profiles.wordpress.org/optimizingmatters/",
    "zytzagoo": "https://profiles.wordpress.org/zytzagoo/",
    "turl": "https://profiles.wordpress.org/turl/"
  }
}
```

## Screenshots

Using Github Workflow to automate the script.

![Github Workflow](/screenshots/get_plugins_r2_workflow-01.png)

![Github Workflow](/screenshots/get_plugins_r2_workflow-02.png)

Example of Cloudflare R2 S3 buckets populated with WordPress plugin zip files and JSON metadata files using S3Browser to view the listings. The example is only using a selection of WordPress plugins I want to mirror for this POC. Script supports full mirror and cache of all >100K plugins.

![Cloudflare R2 S3 Bucket](/screenshots/get_plugins_r2_s3browser-listing-plugins-01.png)

![Cloudflare R2 S3 Bucket](/screenshots/get_plugins_r2_s3browser-listing-plugins-json-metadata-01.png)