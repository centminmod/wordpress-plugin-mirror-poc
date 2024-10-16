# WordPress Plugin Mirror Downloader: User Documentation

## Table of Contents
1. [Introduction](#introduction)
   * [SVN Mirroring](#svn-mirroring)
     * [Using csync2 cluster](#using-csync2-cluster)
   * [Key Components](#key-components)
   * [System Advantages](#system-advantages)
   * [Cloudflare Related Costs](#cloudflare-related-costs)
     * [Cloudflare Workers Dashboard Metrics](#cloudflare-workers-dashboard-metrics)
     * [Cloudflare R2 Dashboard Metrics](#cloudflare-r2-dashboard-metrics)
     * [Cloudflare D1 SQLite Dashboard Metrics](#cloudflare-d1-sqlite-dashboard-metrics)
     * [Cloudflare R2 GraphQL Metrics](#cloudflare-r2-graphql-metrics)
2. [System Overview](#system-overview)
     1.  [Plugin Identification](#1-plugin-identification)
     2.  [API Interaction](#2-api-interaction)
     3.  [Worker Request Generation](#3-worker-request-generation)
     4.  [Worker Processing](#4-worker-processing)
     5.  [R2 Storage Interaction](#5-r2-storage-interaction)
     6.  [Data Compression](#6-data-compression)
     7.  [Response Handling](#7-response-handling)
     8.  [Parallel Execution](#8-parallel-execution)
     9.  [Logging and Metrics](#9-logging-and-metrics)
     10. [Error Handling and Retries](#10-error-handling-and-retries)
     11. [Cache-Only Mode](#11-cache-only-mode)
     12. [Selective Plugin Processing](#12-selective-plugin-processing)
     13. [Plugin Checksum Verification](#13-plugin-checksum-verification)
     14. [Cloudflare D1 SQLite Database Support](#14-cloudflare-d1-sqlite-database-support)
         * [Integration WordPress Plugins Download + R2 S3 Store + D1 SQLite Database](#integration-wordpress-plugins-download--r2-s3-store--d1-sqlite-database)
3. [Examples](#examples)
   * [Mirrored Plugin Checksums](#mirrored-plugin-checksums)
   * [Cached Plugin](#cached-plugin)
   * [wget download speed test](#wget-download-speed-test)
   * [Mirrored WordPress Plugin API End Point](#mirrored-wordpress-plugin-api-end-point)
   * [Github Hosted Plugins](#github-hosted-plugins)
     * [Other WordPress Plugins Not Hosted On Github](#other-wordpress-plugins-not-hosted-on-github)
   * [Demo WordPress Plugin Install Using Local Mirror](#demo-wordpress-plugin-install-using-local-mirror)
     * [Install using default wordpress.org download source](#install-using-default-wordpress.org-download-source)
     * [Installing from local mirror downloaded plugin zip file](#installing-from-local-mirror-downloaded-plugin-zip-file)
     * [Installing from remote mirror server's plugin zip file](#installing-from-remote-mirror-server's-plugin-zip-file)
     * [Demo WordPress Installed Plugin Checksum Verification](#demo-wordpress-installed-plugin-checksum-verification)
4. [Plugin Mirror Screenshots](#plugin-mirror-screenshots)
5. [WordPress Themes Mirror](#wordpress-themes-mirror)
   * [WordPress Themes Cloudflare CDN Benchmarks](#wordpress-themes-cloudflare-cdn-benchmarks)
   * [WordPress Themes Mirrors](#wordpress-themes-mirrors)
     * [WordPress Themes Mirror Screenshots](#wordpress-themes-mirror-screenshots)
     * [WordPress Themes Mirror Cloudflare R2 S3 Metrics Screenshots](#wordpress-themes-mirror-cloudflare-r2-s3-metrics-screenshots)
6. [WordPress Secret Keys API Generator](#wordpress-secret-keys-api-generator)

## Introduction

On October 2, 2024, I began this proof of concept (POC) to develop a solution tailored to my specific needs - [automating WordPress core and plugin installations](#demo-wordpress-plugin-install-using-local-mirror) via [WP-CLI tool](https://wp-cli.org/).  I will continually update this POC as I develop it further. So watch this space.

The WordPress Plugin Mirror Downloader is a robust system designed to efficiently download, cache, and manage [WordPress plugins](#cached-plugin), the [plugin JSON metadata](#mirrored-wordpress-plugin-api-end-point) and [plugin JSON checksum data](#13-plugin-checksum-verification). It optionally supports [Github hosted Wordpress plugins](#github-hosted-plugins) as well. It leverages Cloudflare's edge computing capabilities including CDN, Workers, R2 S3 object storage to provide a high-performance, scalable solution for plugin management. There is also optional [Cloudflare D1 SQLite database support](#14-cloudflare-d1-sqlite-database-support). This project serves as my proof of concept, with the hope that it might inspire others to create their own WordPress.org fallback solutions for their own specific usage requirements, particularly during periods of downtime.

The below example shows that you could setup your own private WordPress plugin mirror with starting costs as low as US$5.35/month and [install your own locally mirrored WordPress plugins](#demo-wordpress-plugin-install-using-local-mirror). You don't have to mirror all WordPress plugins, you can selectively choose to mirror the plugins you use yourself. Or if you're an open source project it would be totally free via [Cloudflare Project Alexandria](https://blog.cloudflare.com/expanding-our-support-for-oss-projects-with-project-alexandria/). Read the below info to see how it works.

The method presented below focuses solely on mirroring and downloading WordPress plugin zip files, rather than mirroring the entire WordPress plugin SVN repository. This is because SVN repositories contain a complete history of all commits and versions for each WordPress plugin, which significantly increases the number of files and the amount of data that needs to be transferred. The way this script works means that you don't need a large amount of local disk storage, as its [cache-only mode](#examples) allows you to directly populate the [Cloudflare R2 S3 object storage bucket](#screenshots), bypassing the need for local downloads if desired. By default, however, local downloads are performed.

To initially populate all 59,979 WordPress plugins into Cloudflare R2 S3 object storage, the process took approximately 25-30 minutes using [automated GitHub Workflow actions](#screenshots). I divided the list of plugins into 29 segments, each handled by a separate GitHub Workflow action that triggered the script in [cache-only mode](#examples), running across 29 Azure compute-based GitHub runner VPS servers with 4 CPU cores each. The script supports parallel mode so set it to use 4 threads for each Github runner server, so 29x4 = 116x CPU threads were utilised to concurrently download and populate the Cloudflare R2 S3 object storage buckets - saving plugin zip file, checksum file and JSON metadata. Github Pro plan only allows up to a max of 40 concurrent Github runner servers at a time so while there is room for further speed up, it has a cap. The plugins were ordered from most popular to least popular, allowing automated runs to prioritize updates for the most frequently used plugins.

The cache-only mode was required due to the limited free disk space on GitHub Workflow's Azure-based VPS runners, as the 59,979 plugins occupied 32GB of space and their JSON metadata files used an additional 420MB. Though cache-only mode now no longer required as I discovered this [Github action](https://github.com/ngocptblaplafla/more-space-plz) which can recover nearly 50GB of disk space from an Azure based Github runner server. Based on [Cloudflare R2 S3 object storage costs](#cloudflare-related-costs) for 33GB of WordPress plugins, for a private WordPress plugin mirror would cost less than **US$0.35/month** though will rise as new plugin versions get downloaded and stored and if under generous R2 Class A/B write and read operation free monthly quota, the R2 operation costs would be free. The Cloudflare Worker costs based on past 24hrs metrics was 765K Worker requests at median of 2.1ms CPU time. So with 100K Worker requests/day free, cost would essentially be the **$5/month** Cloudflare Worker subscription fee as overages won't apply under their generous paid US$5/month subscription plan includes up to 10 million requests per month at average 10ms average CPU time. All up, it would be **US$5.35/month** starting costs which will rise as number of WordPress plugins and Worker and R2 read/write usage increases as outlined [here](#cloudflare-related-costs) and depends on how frequent you plan to [sync the local WordPress plugins mirror](#costs). If you are running a full WordPress mirror for private usage and have hourly syncing, check out my calculated costs based on the past week of doing syncing [here](#costs). If you're only running your own private WordPress plugin mirror, costs will be relatively low. Cloudflare CDN bandwidth is free of charge, so no need to worry about egress bandwidth fees.

Given [Cloudflare CDN cached WordPress plugin download benchmark speeds](#cached-plugin), which showed 43 times faster download speeds and 82% lower latency compared to those served from WordPress.org, it's worth considering that Matt Mullenweg should take up Cloudflare CEO Matthew Prince's [offer of donating capacity to WordPress.org](https://x.com/eastdakota/status/1841154152006627663?t=L0e-TL1cPhkgckxPDG6nvg&s=19). This would dramatically reduce WordPress.org's infrastructure costs and enhance file download speeds. :sunglasses:

Disclaimer: I have been a Cloudflare customer since 2011 and an official Cloudflare community MVP since 2018 (unpaid, similar to the Microsoft MVP program), using Cloudflare's Free, Pro, Business, and Enterprise plans.

### SVN Mirroring

Prior to this POC, I did try POC for SVN mirroring at https://gist.github.com/centminmod/003654673b3c6b11e10edc9353551fd2 and for test 53 WordPress plugins, total disk space to mirror them was approximately 40GB in size. So you will need alot less disk resources and bandwidth if you only focus on WordPress plugin zip files and not the entire SVN repository. In comparison with below mirroring of zip files only, the size for test run of 563 WordPress plugin zip files download and cache into Cloudflare R2 S3 object storage was ~1.27GB in size for zip files and ~18MB for plugin JSON metadata files. Rough maths for 563 plugins taking ~1.3GB storage space. So for 103K plugins would be ~238GB total storage space which will be beyond Cloudflare R2 S3 object storage's Forever Free tier of 10GB/month storage. So there would be additional storage costs - unless you are an open source project under [Cloudflare Project Alexandria](https://blog.cloudflare.com/expanding-our-support-for-oss-projects-with-project-alexandria/).

You can also leverage Cloudflare R2 as mounted Linux FUSE mount via [JuiceFS](https://juicefs.com/docs/community/introduction/) which caches file metadata for better performance and allows you to mount Cloudflare R2 S3 mounts in sharded mounts as well. See my write up and benchmarks for Cloudflare R2 + JuiceFS https://github.com/centminmod/centminmod-juicefs.

Example of a JuiceFS FUSE mount on AlmaLinux based Linux server using Cloudflare R2 S3 object storage. The system would just see it as a regular mount for storage. Just be sure when creating Cloudflare R2 buckets you use location hints to ensure the R2 buckets are closest to your intended servers with the JuiceFS FUSE mount.

```
df -hT /home/juicefs_mount
Filesystem        Type          Size  Used Avail Use% Mounted on
JuiceFS:myjuicefs fuse.juicefs  1.0P     0  1.0P   0% /home/juicefs_mount
```

#### Using csync2 cluster.

Recently I been working on custom RPM package builds for newer featured forked csync2 2.1.1 for AlmaLinux/Rocky Linux which have atomic file updates, nanoseconds support and parallel node updates via inotifywait triggers as csync2 2.0 is too old. I could setup multiple csync2 2.1.1 server nodes each setup with with HTTP forward proxies + proxychains proxy rotation and have servers configured to clone the WordPress SVN repos from any of the csync2 node servers and have updates automatically sync to all csync2 nodes. It would scale easily as you add new csync2 node servers to cluster, your HTTP forward proxies pool for proxychains would scale up too.

WordPress SVN repos:

* https://core.svn.wordpress.org/
* https://plugins.svn.wordpress.org/
* https://themes.svn.wordpress.org/

### Key Components

1. **Cloudflare Worker**: A serverless JavaScript function that acts as an intermediary between the client (bash script) and the data sources (WordPress.org and Cloudflare R2 storage). See https://developers.cloudflare.com/workers/ and https://developers.cloudflare.com/workers/tutorials/. And how Cloudflare Workers have bindings to other Cloudflare products like Cloudflare R2 S3 object storage https://developers.cloudflare.com/workers/runtime-apis/bindings/.

2. **Cloudflare R2 Storage**: An S3-compatible object storage system used to cache [plugin ZIP files](#cached-plugin), [ZIP file checksums](#mirrored-plugin-checksums) and [plugin JSON metadata](#mirrored-wordpress-plugin-api-end-point). Cloudflare R2 S3 object storage has free egress bandwidth costs so you only pay for object storage and read/writes to object storage.

3. **Cloudflare DQ SQLite Database (optional)**: Optional [Cloudflare D1 SQLite databases](#14-cloudflare-d1-sqlite-database-support) for adding and querying the WordPress plugins' JSON metadata and JSON checksum information.

4. **Bash Script**: A local client that orchestrates the plugin download process, interacts with the WordPress API, and communicates with the Cloudflare Worker.

### System Advantages

- **Efficient Caching**: By utilizing Cloudflare R2 storage, the system significantly reduces the load on WordPress servers and improves download speeds for frequently requested plugins. See [Cloudflare CDN cached plugin benchmarks](#cached-plugin).

- **Version Tracking**: The system maintains a local record of installed plugin versions, enabling selective updates and reducing unnecessary downloads.

- **Parallel Processing**: The bash script supports concurrent downloads, dramatically reducing the time required for bulk plugin updates.

- **Comprehensive Logging**: Detailed logging at both the Worker and bash script levels facilitates troubleshooting and performance optimization.

- **Advanced Error Handling**: Robust error handling mechanisms in both the Worker and bash script ensure graceful failure recovery and informative error reporting.

- **Rate Limiting**: Implemented at the Worker level to prevent abuse and ensure fair usage of resources.

- **Compression Support**: The Worker supports gzip compression, reducing bandwidth usage and improving download speeds.

- **Separate Caching for ZIP, JSON, and Checksums**: By caching ZIP files, JSON metadata, and checksums separately, the system can efficiently handle partial updates and reduce storage costs.

- **Cache-Only Mode**: A new feature allows checking and updating the cache and R2 bucket without downloading files, useful for preemptive caching and system checks.

- **Checksum Verification**: The system now fetches and stores plugin checksums, enabling integrity verification of downloaded files.

- **Selective Plugin Processing**: New `-s` and `-e` options allow processing a specific range of plugins when used with the `-a` flag, enabling parallel processing of different plugin ranges.

<a name="bridge"></a>
- **WordPress Plugin 1.2 API Bridge Worker**: An additional WordPress Plugin API Bridge Worker is created using a separate Cloudflare Worker. It is designed to bridge the gap between the WordPress Plugin API 1.0 and 1.2 versions `https://api.wordpress.org/plugins/info/1.0` vs `https://api.wordpress.org/plugins/info/1.2`. It allows clients to query plugin information using the 1.2 API format while fetching data from either a mirrored 1.0 API endpoint or the official WordPress.org 1.0 API, providing flexibility and reliability in data retrieval. This bridge worker eliminates the need for me to have any sort of database hosted as API 1.2 would just rely on the already mirrored API 1.0 JSON metadata stored in Cloudflare R2 object storage.

  ```bash
  curl -s -H "Accept: application/json" "https://api.mycloudflareproxy_domain.com/plugins/info/1.2/?action=plugin_information&slug=autoptimize&locale=en_US" | jq -r '[.name, .slug, .version, .download_link, .tested, .requires_php]'
  [
    "Autoptimize",
    "autoptimize",
    "3.1.12",
    "https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip",
    "6.6.2",
    "5.6"
  ]
  ```

- **Cloudflare D1 SQLite Database Support**: Optional [Cloudflare D1 SQLite database support](#14-cloudflare-d1-sqlite-database-support) via a separate Cloudflare Worker binded to Cloudflare R2 buckets and D1 SQLite database for adding WordPress plugins' JSON metadata and JSON checksum information. You can track your [Cloudflare D1 SQLite dashboard metrics](#cloudflare-d1-sqlite-dashboard-metrics) as well.

### Cloudflare Related Costs

Cloudflare CDN is free of charge, so only costs will be related to using Cloudflare Workers and Cloudflare R2 S3 object storage as outlined below. Optionally, you can also use [Cloudflare D1 SQLite databases](#14-cloudflare-d1-sqlite-database-support). If you're only running your own private WordPress plugin mirror, costs will be relatively low compared to the 4 cost examples I outlined further below. I doubt it would cost more than US$5.35/month to begin with.

For Cloudflare Workers pricing https://developers.cloudflare.com/workers/platform/pricing/.

| Tier | Requests¹ ² | Duration | CPU time |
|------|-------------|----------|----------|
| Free | 100,000 per day | No charge for duration | 10 milliseconds of CPU time per invocation |
| Standard | 10 million included per month<br>+$0.30 per additional million | No charge or limit for duration | 30 million CPU milliseconds included per month<br>+$0.02 per additional million CPU milliseconds<br>Max of 30 seconds of CPU time per invocation<br>Max of 15 minutes of CPU time per [Cron Trigger](https://developers.cloudflare.com/workers/configuration/cron-triggers/) or [Queue Consumer](https://developers.cloudflare.com/queues/configuration/javascript-apis/#consumer) invocation |

¹ Inbound requests to your Worker. Cloudflare does not bill for [subrequests](https://developers.cloudflare.com/workers/platform/limits/#subrequests) you make from your Worker.
² Requests to static assets are free and unlimited.

For Cloudflare R2 pricing https://developers.cloudflare.com/r2/platform/pricing/ and calculator at https://r2-calculator.cloudflare.com/.

Cloudflare R2 free plan quota pricing and PAYGO pricing beyond free plan.

| Feature | Forever Free | Standard Storage | Infrequent Access Storage (Beta) |
|---------|--------------|-------------------|----------------------------------|
| Storage | 10 GB / month | $0.015 / GB-month | $0.01 / GB-month |
| Class A operations: mutate state | 1,000,000 / month | $4.50 / million requests | $9.00 / million requests |
| Class B operations: read existing state | 10,000,000 / month | $0.36 / million requests | $0.90 / million requests |
| Data Retrieval (processing) | N/A | None | $0.01 / GB |
| Egress (data transfer to Internet) | N/A | Free | Free |

For [Cloudflare D1 SQLite pricing](https://developers.cloudflare.com/d1/platform/pricing/) which is optional and not really required for my particular usage of WordPress plugin mirror system. But if you want to use it check it out [here](#14-cloudflare-d1-sqlite-database-support).

| Feature | Workers Free | Workers Paid |
|---------|--------------|--------------|
| Rows read | 5 million / day | First 25 billion / month included + $0.001 / million rows |
| Rows written | 100,000 / day | First 50 million / month included + $1.00 / million rows |
| Storage (per GB stored) | 5 GB (total) | First 5 GB included + $0.75 / GB-mo |

Note:
- Free limits reset daily at 00:00 UTC.
- Monthly included limits for paid plans reset based on your monthly subscription renewal date.
- There are no data transfer (egress) or throughput (bandwidth) charges for data accessed from D1.
- Row size or the number of columns in a row does not impact how rows are counted.
- DDL operations may contribute to a mix of read rows and write rows.
- Indexes will add an additional written row when writes include the indexed column.

<a name="costs"></a>
**October 10, 2024:** It's been a week since I created this POC and had time to do some metrics and cost calculations to have a better idea of operational costs for a full private self usage WordPress plugin mirror syncing if you are concerned with up to date WordPress plugin versions etc. Note these costs will be alot less if you choose only to select your few WordPress plugins you want to use. My previous estimated costs can still be seen [here](#morecosts) and you can see a table breaking down how many syncs you could do per month for various monthly budgets [here](#synccosts).

There are ~60K active opened WordPress plugins with latest version taking up a total of ~35GB of disk space. If I do hourly syncs of the mirror, these would be the estimated Cloudflare Worker and R2 costs below for each month - for 24x30 = 720 syncs/month. R2 disk usage would increase over time though as new WordPress plugins and version updates are added. Cloudflare CDN bandwidth is free always so not applicable in cost calculations.

1. R2 Storage Costs
   - Storage: 35 GB at $0.015 per GB-month
   - First 10GB free = $0
   - 25 * $0.015 = $0.375 per month

2. R2 Operations
   - Write operations (Class A): from Cloudflare R2 metrics each sync with -f forced flag to bypass CDN cache, is making 50K Class A writes to R2 bucket. For hourly syncing, that comes to 50k x 24 x 30 = 36 million R2 Class A writes
   - First 1 million writes free = $0
   - 35 * $4.50 = $157.50 per month
   - Read operations (Class B): from Cloudflare R2 metrics each sync with -f forced flag to bypass CDN cache, is making 100K Class B reads to R2 bucket. For hourly syncing, that comes to 100k x 24 x 30 = 72 million R2 Class B reads
   - First 10 million writes free = $0
   - 62 * $0.36 = $22.32 per month

3. Cloudflare Worker
   - Requests: from Cloudflare Worker metrics each sync, is making 250K requests at 2.7ms medium CPU time. For hourly syncing, that comes to 250k x 24 x 30 = 180 million requests
    - First 10 million included in Standard tier
    - Additional 170 million at $0.30 per million
    - 170 * $0.30 = $51.00 per month
   - CPU time: 180 million * 2.7ms = 486 million CPU milliseconds
    - First 30 million CPU milliseconds included
    - Additional 150 million at $0.02 per million
    - 150 * $0.02 = $3.00 per month
   - $5/month subscription fee

Total Cost Breakdown:

- R2 Storage: $0.375
- R2 Write Operations: $157.50
- R2 Read Operations: $22.32 (reduced pricing if implement Cloudflare CDN cache using [Cache Rules](https://developers.cloudflare.com/cache/how-to/cache-rules/) in front of R2 stored files)
- Cloudflare Worker Requests: $51.00
- Cloudflare Worker CPU time: $3
- Cloudflare Worker Subscription fee: $5.00

Total Monthly Cost: $239.20 per month

<a name="synccosts"></a>
The below table provides a good overview of how many syncs you can perform at different budget levels where the limiting factor is R2 Class A write operations costs for free tier.

Based on following assumptions:

1. R2 Storage: $0.375/month (fixed for 35GB, 10GB free)
2. R2 Write Operations (Class A): 
   - Free: 1 million/month
   - Beyond: $4.50 per million
3. R2 Read Operations (Class B):
   - Free: 10 million/month
   - Beyond: $0.36 per million
4. Cloudflare Worker Requests:
   - Free: 10 million/month
   - Beyond: $0.30 per million
5. Cloudflare Worker CPU time:
   - Free: 30 million ms/month
   - Beyond: $0.02 per million ms
6. Cloudflare Worker Subscription fee: $5/month (fixed)

So if each sync uses:

- R2 Write Operations: 50,000
- R2 Read Operations: 100,000
- Worker Requests: 250,000
- Worker CPU time: 675,000 ms

If you can live with syncing once per day, cost would start out at $10/month with R2 object storage costs being the variable component. These are costs if you are mirroring and syncing all >60K WordPress plugins. Costs would be less if you are only mirroring and syncing your selective WordPress plugin list.

| Budget | Syncs per Month | Limiting Factor | Approximate Sync Frequency |
|--------|-----------------|-----------------|------------------------|
| Free Tier | 20 | R2 Class A operations | Every 36 hours |
| $10/month | 29 | R2 Class A operations | Every 24.8 hours |
| $20/month | 53 | R2 Class A operations | Every 13.6 hours |
| $25/month | 65 | R2 Class A operations | Every 11 hours |
| $30/month | 77 | R2 Class A operations | Every 9.4 hours |
| $40/month | 101 | R2 Class A operations | Every 7.1 hours |
| $50/month | 125 | R2 Class A operations | Every 5.8 hours |
| $60/month | 149 | R2 Class A operations | Every 4.8 hours |
| $75/month | 185 | R2 Class A operations | Every 3.9 hours |
| $100/month | 245 | R2 Class A operations | Every 2.9 hours |
| $125/month | 305 | R2 Class A operations | Every 2.4 hours |
| $150/month | 365 | R2 Class A operations | Every 2 hours |
| $200/month | 485 | R2 Class A operations | Every 1.5 hours |
| $250/month | 725 | R2 Class A operations | Every 59.6 minutes |

#### Other Cost Estimations

<a name="morecosts"></a>
<details>
<summary>Click to expand other estimated cost calculations</summary>

#### Example 1 - cost calculation for:

* 250GB of R2 storage with 5 million write and 25 million read operations 
* Cloudflare Worker for 10 million requests averaging 1.8ms CPU time

1. R2 Storage Costs:
   - Storage: 250 GB at $0.015 per GB-month
   - First 10GB free = $0
   - 240 * $0.015 = $3.60 per month

2. R2 Operations:
   - Write operations (Class A): 5 million at $4.50 per million
   - 5 * $4.50 = $22.50 per month
   - Read operations (Class B): 25 million at $0.36 per million
   - 25 * $0.36 = $9.00 per month

3. Cloudflare Worker:
   - Requests: 10 million (included in Standard tier)
   - CPU time: 10 million * 1.8ms = 18 million CPU milliseconds (within the 30 million included)
   - $5/month subscription fee

Total cost breakdown:

- R2 Storage: $3.60
- R2 Write Operations: $22.50
- R2 Read Operations: $9.00
- Cloudflare Worker usage: $0 (within included limits)
- Cloudflare Worker Subscription fee: $5

Total monthly cost: $40.10 per month

#### Example 2 - large cost calculation for:

- 2500GB of R2 storage
- 50 million write operations and 250 million read operations on R2
- Cloudflare Worker handling 100 million requests, averaging 2.5ms CPU time per request

1. R2 Storage Costs
   - Storage: 2500 GB at $0.015 per GB-month
   - First 10GB free = $0
   - 2490 * $0.015 = $37.35 per month

2. R2 Operations
   - Write operations (Class A): 50 million at $4.50 per million
   - 50 * $4.50 = $225.00 per month
   - Read operations (Class B): 250 million at $0.36 per million
   - 250 * $0.36 = $90.00 per month

3. Cloudflare Worker
   - Requests: 100 million
    - First 10 million included in Standard tier
    - Additional 90 million at $0.30 per million
    - 90 * $0.30 = $27.00 per month
   - CPU time: 100 million * 2.5ms = 250 million CPU milliseconds
    - First 30 million CPU milliseconds included
    - Additional 220 million at $0.02 per million
    - 220 * $0.02 = $4.40 per month
   - $5/month subscription fee

Total Cost Breakdown:

- R2 Storage: $37.35
- R2 Write Operations: $225.00
- R2 Read Operations: $90.00
- Cloudflare Worker Requests: $27.00
- Cloudflare Worker CPU time: $4.40
- Cloudflare Worker Subscription fee: $5.00

Total Monthly Cost: $388.75 per month

#### Example 3 - extra large cost calculation for:

- 2000GB of R2 storage
- 500 million write operations and 250 million read operations on R2
- Cloudflare Worker handling 1 billion requests, averaging 3ms CPU time per request

1. R2 Storage Costs
   - Storage: 5000 GB at $0.015 per GB-month
   - First 10GB free = $0
   - 4990 * $0.015 = $74.85 per month

2. R2 Operations
   - Write operations (Class A): 500 million at $4.50 per million
   - 500 * $4.50 = $2,250.00 per month
   - Read operations (Class B): 2.5 billion at $0.36 per million
   - 2500 * $0.36 = $900.00 per month

3. Cloudflare Worker
   - Requests: 1 billion
    - First 10 million included in Standard tier
    - Additional 990 million at $0.30 per million
    - 990 * $0.30 = $297.00 per month
   - CPU time: 1 billion * 3ms = 3 billion CPU milliseconds
    - First 30 million CPU milliseconds included
    - Additional 2,970 million at $0.02 per million
    - 2970 * $0.02 = $59.40 per month
   - $5/month subscription fee

Total Cost Breakdown:

- R2 Storage: $74.85
- R2 Write Operations: $2,250.00
- R2 Read Operations: $900.00
- Cloudflare Worker Requests: $297.00
- Cloudflare Worker CPU time: $59.40
- Cloudflare Worker Subscription fee: $5.00

Total Monthly Cost: $3,586.25 per month

#### Example 4 - more realistic R2 Writes:

There are ~103K Wordpress plugins as of writing in SVN plugin repository. Though only 59,979 of these plugins are opened and published and not in `closed` state. The 59,979 plugins take up 32GB of space and their JSON metadata files takes up 420MB of space. If calculate it on higher of the two counts and each plugin would consistently release a new version per day for 30 days, there would be 2x 103,000 R2 writes/day - one for R2 write for zip file and one for R2 write for JSON metadata file = 206,000/day = 6.18 million R2 writes per month. Obviously, not every plugin would be releasing a new version every day for an entire month.

- 250GB of R2 storage with 6.18 million write and 10 billion read operations. Note, if you implement Cloudflare CDN cache using [Cache Rules](https://developers.cloudflare.com/cache/how-to/cache-rules/) in front of R2 stored files, you won't get anywhere near 10 bliion read operations in reality. Example of Cloudflare CDN cached mirrored WordPress plugin [here](#cached-plugin).
- Cloudflare Worker handling 10 billion requests, averaging 3ms CPU time per request
- R2 read and write operations also haven't accounted for my script's inspection of the R2 buckets' respective contents.

1. R2 Storage Costs
   - Storage: 250 GB at $0.015 per GB-month
   - First 10GB free = $0
   - 240 * $0.015 = $3.60 per month

2. R2 Operations
   - Write operations (Class A): 6.18 million at $4.50 per million
   - 6.18 * $4.50 = $27.81 per month
   - Read operations (Class B): 10 billion at $0.36 per million
   - 10000 * $0.36 = $3,600.00 per month

3. Cloudflare Worker
   - Requests: 10 billion
    - First 10 million included in Standard tier
    - Additional 9990 million at $0.30 per million
    - 9990 * $0.30 = $2,997.00 per month
   - CPU time: 10 billion * 3ms = 30 billion CPU milliseconds
    - First 30 million CPU milliseconds included
    - Additional 29,970 million at $0.02 per million
    - 29970 * $0.02 = $599.40 per month
   - $5/month subscription fee

Total Cost Breakdown:

- R2 Storage: $3.60
- R2 Write Operations: $27.81
- R2 Read Operations: $3,600.00 (reduced pricing if implement Cloudflare CDN cache using [Cache Rules](https://developers.cloudflare.com/cache/how-to/cache-rules/) in front of R2 stored files)
- Cloudflare Worker Requests: $2,997.00
- Cloudflare Worker CPU time: $599.40
- Cloudflare Worker Subscription fee: $5.00

Total Monthly Cost: $7,232.81 per 
</details>

#### Cloudflare Workers Dashboard Metrics

Here's the mirror's primary Cloudflare Workers' dashboard metrics. Includes all the testing I have done for the week - though only started October 2, 2024.

![Cloudflare Workers Dashboard metrics](/screenshots/cf-workers-dashboard-01.png)

#### Cloudflare R2 Dashboard Metrics

Full WordPress plugin mirror populating just under 60K plugins and a few new versions of plugins included account = 35GB of storage used right now. You can see the Class A (writes) and Class B (read) metrics as well. This is only for R2 bucket for plugin zip files and plugin checksum files. A separate R2 bucket is used for plugin JSON metadata.

![Cloudflare R2 Dashboard metrics](/screenshots/cf-r2-dashboard-01.png)

Shows gradual increase in R2 bucket storage usage

![Cloudflare R2 Dashboard metrics](/screenshots/cf-r2-dashboard-02.png)

Class A writes

![Cloudflare R2 Dashboard metrics](/screenshots/cf-r2-dashboard-03.png)

Class B reads

![Cloudflare R2 Dashboard metrics](/screenshots/cf-r2-dashboard-04.png)

#### Cloudflare D1 SQLite Dashboard Metrics

Full WordPress plugin mirror populating approximately 60K plugins's Cloudlare R2 stored JSON metadata and JSON checksum data into a Cloudflare D1 SQLite database ended up occupying 2.04GB of disk space. The script I ran with 8 parallel threads with each thread processing 200 batch scan and insertions into D1 SQLite database as outlined [here](#14-cloudflare-d1-sqlite-database-support). I am still figuring out Cloudflare D1 SQLite's [rate limits](https://developers.cloudflare.com/d1/platform/limits/).

```bash
> SELECT COUNT(*) FROM plugins;
COUNT(*)
60137

Response time 1973ms, query time 1.25ms
```

Queries

![Cloudflare D1 SQLite Dashboard metrics](/screenshots/cf-d1-dashboard-01.png)

Latencies

![Cloudflare D1 SQLite Dashboard metrics](/screenshots/cf-d1-dashboard-02.png)

Row Metrics

![Cloudflare D1 SQLite Dashboard metrics](/screenshots/cf-d1-dashboard-03.png)

#### Cloudflare R2 GraphQL Metrics

We can also inspect [Cloudflare R2 metrics](https://developers.cloudflare.com/r2/platform/metrics-analytics/) via their [Cloudflare GraphQL API](https://developers.cloudflare.com/analytics/graphql-api/) for R2 read/writes (`GetObject`/`PutObject`).

```bash
export ACCOUNT_ID='YOUR_CLOUDFLARE_ACCOUNT_ID'
export API_TOKEN='YOUR_CF_API_TOKEN'
export BUCKET_NAME='YOUR_R2_BUCKET_NAME'
```

```bash
./query_r2_graphql.sh
GraphQL query saved to r2_graphql_query.graphql
Checking last 30 minutes...
Metrics found!

Checking last 60 minutes...
Metrics found!

Checking last 120 minutes...
Metrics found!

Checking last 360 minutes...
Metrics found!

Checking last 720 minutes...
Metrics found!

Checking last 1440 minutes...
Metrics found!

Checking last 2880 minutes...
Metrics found!

Checking last 4320 minutes...
Metrics found!

Comprehensive Summary:

Last 30 minutes:
Querying data from 2024-10-04T05:41:23Z to 2024-10-04T06:11:23Z
GetObject: 4

Last 60 minutes:
Querying data from 2024-10-04T05:11:23Z to 2024-10-04T06:11:23Z
GetObject: 475
PutObject: 240

Last 120 minutes:
Querying data from 2024-10-04T04:11:24Z to 2024-10-04T06:11:24Z
GetObject: 1121
PutObject: 570

Last 360 minutes:
Querying data from 2024-10-04T00:11:24Z to 2024-10-04T06:11:24Z
GetObject: 1149
PutObject: 579

Last 720 minutes:
Querying data from 2024-10-03T18:11:25Z to 2024-10-04T06:11:25Z
GetObject: 1149
PutObject: 579

Last 1440 minutes:
Querying data from 2024-10-03T06:11:25Z to 2024-10-04T06:11:25Z
GetObject: 1152
PutObject: 580

Last 2880 minutes:
Querying data from 2024-10-02T06:11:25Z to 2024-10-04T06:11:25Z
GetObject: 2596
PutObject: 891

Last 4320 minutes:
Querying data from 2024-10-01T06:11:26Z to 2024-10-04T06:11:26Z
GetObject: 3396
PutObject: 1149

Conclusion:
Metrics were found in at least one time range.
Most recent activity: Last 30 minutes

Activity summary:
  PutObject: 1149
  GetObject: 3396
  Querying data from 2024-10-01T06:11:26Z to 2024-10-04T06:11:26Z
- Last 4320 minutes:
  PutObject: 891
  GetObject: 2596
  Querying data from 2024-10-02T06:11:25Z to 2024-10-04T06:11:25Z
- Last 2880 minutes:
  PutObject: 580
  GetObject: 1152
  Querying data from 2024-10-03T06:11:25Z to 2024-10-04T06:11:25Z
- Last 1440 minutes:
  PutObject: 579
  GetObject: 1149
  Querying data from 2024-10-03T18:11:25Z to 2024-10-04T06:11:25Z
- Last 720 minutes:
  PutObject: 579
  GetObject: 1149
  Querying data from 2024-10-04T00:11:24Z to 2024-10-04T06:11:24Z
- Last 360 minutes:
  PutObject: 570
  GetObject: 1121
  Querying data from 2024-10-04T04:11:24Z to 2024-10-04T06:11:24Z
- Last 120 minutes:
  PutObject: 240
  GetObject: 475
  Querying data from 2024-10-04T05:11:23Z to 2024-10-04T06:11:23Z
- Last 60 minutes:
  GetObject: 4
  Querying data from 2024-10-04T05:41:23Z to 2024-10-04T06:11:23Z
- Last 30 minutes:

Trend analysis:
Significant activity in the last 24 hours. Review the 24-hour metrics for a comprehensive overview.
Activity detected in the last 72 hours. Compare with 24-hour and 48-hour metrics to identify trends.
```

The above results are just after a Github Workflow automated run downloading and caching locally 563 WordPress plugins. Server uses UTC timezone. But last 360 minutes seems to be close to the Cloudflare R2 read/write usage for the run:

```bash
- Last 360 minutes:
  PutObject: 570
  GetObject: 1121
  Querying data from 2024-10-04T04:11:24Z to 2024-10-04T06:11:24Z
```

The `r2_graphql_query.graphql` query used - adjusting the `START_DATE` and `END_DATE` for 30, 60, 120, 360, 720, 1440, 2880, 4320 durations in minutes.

```graphql
query R2ReadWriteOperations($ACCOUNT_ID: String!, $START_DATE: Time!, $END_DATE: Time!, $BUCKET_NAME: String) {
  viewer {
    accounts(filter: { accountTag: $ACCOUNT_ID }) {
      r2OperationsAdaptiveGroups(
        limit: 10000
        filter: {
          datetime_geq: $START_DATE
          datetime_leq: $END_DATE
          bucketName: $BUCKET_NAME
          actionType_in: ["GetObject", "PutObject", "CreateMultipartUpload", "UploadPart", "CompleteMultipartUpload"]
        }
      ) {
        sum {
          requests
        }
        dimensions {
          actionType
        }
      }
    }
  }
}
```

More recent R2 metrics from attempts at full WordPress plugin mirroring and populating R2 object storage:

```bash
./query_r2_graphql.sh
GraphQL query saved to r2_graphql_query.graphql
Checking last 30 minutes...
No metrics found.

Checking last 60 minutes...
No metrics found.

Checking last 120 minutes...
No metrics found.

Checking last 360 minutes...
Metrics found!

Checking last 720 minutes...
Metrics found!

Checking last 1440 minutes...
Metrics found!

Checking last 2880 minutes...
Metrics found!

Checking last 4320 minutes...
Metrics found!

Comprehensive Summary:

Last 30 minutes:
No metrics found

Last 60 minutes:
No metrics found

Last 120 minutes:
No metrics found

Last 360 minutes:
Querying data from 2024-10-05T08:23:38Z to 2024-10-05T14:23:38Z
GetObject: 48245
PutObject: 11168

Last 720 minutes:
Querying data from 2024-10-05T02:23:38Z to 2024-10-05T14:23:38Z
GetObject: 201650
PutObject: 28195

Last 1440 minutes:
Querying data from 2024-10-04T14:23:38Z to 2024-10-05T14:23:38Z
GetObject: 464787
PutObject: 111423

Last 2880 minutes:
Querying data from 2024-10-03T14:23:39Z to 2024-10-05T14:23:39Z
GetObject: 469632
PutObject: 114657

Last 4320 minutes:
Querying data from 2024-10-02T14:23:39Z to 2024-10-05T14:23:39Z
GetObject: 470208
PutObject: 114665

Conclusion:
Metrics were found in at least one time range.
Most recent activity: Last 360 minutes

Activity summary:
- Last 4320 minutes:
  Querying data from 2024-10-02T14:23:39Z to 2024-10-05T14:23:39Z
  GetObject: 470208
  PutObject: 114665
- Last 2880 minutes:
  Querying data from 2024-10-03T14:23:39Z to 2024-10-05T14:23:39Z
  GetObject: 469632
  PutObject: 114657
- Last 1440 minutes:
  Querying data from 2024-10-04T14:23:38Z to 2024-10-05T14:23:38Z
  GetObject: 464787
  PutObject: 111423
- Last 720 minutes:
  Querying data from 2024-10-05T02:23:38Z to 2024-10-05T14:23:38Z
  GetObject: 201650
  PutObject: 28195
- Last 360 minutes:
  Querying data from 2024-10-05T08:23:38Z to 2024-10-05T14:23:38Z
  GetObject: 48245
  PutObject: 11168

Trend analysis:
No activity in the last hour. Consider investigating if this is unexpected.
Significant activity in the last 24 hours. Review the 24-hour metrics for a comprehensive overview.
Activity detected in the last 72 hours. Compare with 24-hour and 48-hour metrics to identify trends.
```

## System Overview

The WordPress Plugin Mirror Downloader operates through a series of coordinated steps involving the bash script, Cloudflare Worker, and R2 storage. Here's a detailed breakdown of the process:

### 1. **Plugin Identification**:
- The bash script reads a list of desired plugins from a configuration file or command-line arguments. It can also be set to read from a WordPress installation's plugin directory the list of WordPress plugins to download. It also reads and detects [Github hosted Wordpress plugins](#github-hosted-plugins) and mirrors them appropriately.
- It compares this list against a local cache of installed plugins and their versions.
- The script identifies plugins that need to be downloaded or updated based on version discrepancies.

### 2. **API Interaction**:
- For each plugin requiring action, the script queries the WordPress.org API to fetch the latest version information and download URL.
- This step ensures that the system always works with the most up-to-date plugin data.

### 3. **Worker Request Generation**:
- The script constructs HTTP requests to the Cloudflare Worker for both the plugin ZIP file, the respective checksum file and its JSON metadata.
- These requests include query parameters specifying the plugin name, version, and desired content type (ZIP, checksum file or JSON).

### 4. **Worker Processing**:
- Upon receiving a request, the Worker first checks if the requested data exists in the appropriate R2 bucket (WP_PLUGIN_STORE for ZIPs, WP_PLUGIN_INFO for JSONs).
- If the data is found in R2, the Worker serves it directly, incrementing the `r2Hits` metric.
- If not found, the Worker fetches the data from WordPress.org, stores it in R2, and then serves it. This process increments the `wordpressFetches` and `cacheMisses` metrics.

### 5. **R2 Storage Interaction**:
- The Worker interacts with R2 storage using the `env.WP_PLUGIN_STORE` and `env.WP_PLUGIN_INFO` bindings.
- For storage, the Worker uses `put` operations with appropriate metadata.
- For retrieval, it uses `get` operations, falling back to WordPress.org if the object is not found.

### 6. **Data Compression**:
- The Worker applies gzip compression to the response if the client supports it, as indicated by the `Accept-Encoding` header.
- This compression is performed using the `CompressionStream` API.

### 7. **Response Handling**:
- The bash script receives the Worker's response, which includes the requested data (ZIP , checksum file or JSON) and relevant headers.
- It processes this response, saving the data to the local filesystem and updating its version tracking information.

### 8. **Parallel Execution**:
- When configured for parallel processing, the bash script uses `xargs` to spawn multiple instances of the download process.
- This parallelization significantly reduces the total time required for bulk plugin updates.

### 9. **Logging and Metrics**:
- Throughout the process, both the Worker and bash script maintain detailed logs.
- The Worker tracks key metrics such as cache hits, WordPress fetches, and error counts.
- The bash script logs each step of its operation, including version checks, download attempts, and file operations.

### 10. **Error Handling and Retries**:
- The system implements multiple layers of error handling, including network retries in the Worker and fallback mechanisms in the bash script.
- Detailed error messages are propagated from the Worker to the bash script, allowing for informed troubleshooting.

### 11. **Cache-Only Mode**:
- When activated, this mode allows the system to check and update the cache and R2 bucket without downloading files.
- It's useful for preemptive caching, system checks, and reducing unnecessary downloads.

### 12. **Selective Plugin Processing**:
- The `-s` and `-e` options, when used with `-a`, allow you to process a specific range of plugins from the SVN list. This feature is useful for:

  - Distributing the workload across multiple machines or processes.
  - Focusing on a specific subset of plugins for testing or analysis.
  - Resuming a partially completed download process from a specific point.

  To use this feature effectively:

  1. Determine the total number of plugins in the SVN list.
  2. Divide the total range into smaller chunks based on your processing capacity.
  3. Run multiple instances of the script, each with a different range.

  Example of running three parallel processes:

  ```bash
  # Terminal 1
  ./get_plugins_r2.sh -a -s 1 -e 1000 -p 4 -d

  # Terminal 2
  ./get_plugins_r2.sh -a -s 1001 -e 2000 -p 4 -d

  # Terminal 3
  ./get_plugins_r2.sh -a -s 2001 -e 3000 -p 4 -d
  ```

  Example output running 2 parallel threads in debug and cache only modes processing all SVN plugin list but for only first 4 plugins on the list:

  ```bash
  ./get_plugins_r2.sh -p 2 -d -c -a -s 1 -e 4
  Processing plugins from line 1 to 4
  Loaded 4 plugins.
  Running in parallel with 2 jobs...
  ```

### 13. **Plugin Checksum Verification**:
- The system now includes functionality to fetch, store, and verify plugin checksums. This new feature enhances security by allowing users to verify the integrity of downloaded plugin files against the official WordPress.org checksums.

#### How it works:

1. The system fetches official checksums from WordPress.org for each plugin.
2. Checksums are stored in the R2 bucket alongside plugin files and metadata.
3. Users can verify downloaded plugins against these checksums to ensure file integrity.

#### Benefits:

- Detect potentially tampered or corrupted plugin files
- Enhance overall security of the plugin management process
- Provide an additional layer of verification for cached plugins

#### Usage:

To manually verify a plugin's checksums:

1. Fetch the checksums:
   ```bash
   curl -s "https://your-worker-url.workers.dev?plugin=plugin-name&version=version&type=checksums" > checksums.json
   ```

2. Use a tool like `jq` to parse the JSON and compare checksums:
   ```bash
   jq -r '.files[] | "\(.md5)  \(.filename)"' checksums.json > checksums.md5
   md5sum -c checksums.md5
   ```

#### Verification Of Plugin Checksums Against Official WordPress Version

As we are mirroring and caching WordPress's plugins checksum JSON data file, we can also compare our locally mirrored and cached copy against WordPress official WordPress plugin's checksum JSON data - making sure everything is legit and that you are indeed download the exact same WordPress plugin zip file as from official WordPress site when we [rely on the checksum data](#demo-wordpress-installed-plugin-checksum-verification).

A simple diff check against locally mirrored and cached copy of `classic-editor` checksum JSON data file `https://downloads.mycloudflareproxy_domain.com/plugin-checksums/classic-editor/1.6.5.json` against official WordPress copy of checksum JSON data file `https://downloads.wordpress.org/plugin-checksums/classic-editor/1.6.5.json`:

```bash
diff -u <(curl -s https://downloads.wordpress.org/plugin-checksums/classic-editor/1.6.5.json | jq -r) <(curl -s https://downloads.mycloudflareproxy_domain.com/plugin-checksums/classic-editor/1.6.5.json | jq -r)
```
```diff
--- /dev/fd/63  2024-10-07 23:05:14.225476299 +0000
+++ /dev/fd/62  2024-10-07 23:05:14.225476299 +0000
@@ -20,5 +20,6 @@
       "md5": "336c5ec12b70f9bb30d6b917fdc04a56",
       "sha256": "44694da8b97d4e5b49cd8b678154c4078bb1ffef00ac37d09a593d26b8e8365d"
     }
-  }
+  },
+  "zip_mirror": "https://downloads.mycloudflareproxy_domain.com/classic-editor.1.6.5.zip"
 })
```

As you can see all the plugin's checksums are identical with only difference being in my locally mirrored and cached copy I also add `zip_mirror` link to the mirrored WordPress plugin's zip download file.

The same verification can be done for locally mirrored and cached WordPress plugin's JSON metadata file `https://api.mycloudflareproxy_domain.com/plugins/info/1.0/classic-editor.json` comparing it to official WordPress JSON metadata file `https://api.wordpress.org/plugins/info/1.0/classic-editor.json`:

```bash
diff -u <(curl -s https://api.wordpress.org/plugins/info/1.0/classic-editor.json | jq 'del(.sections, .screenshots)') <(curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/classic-editor.json | jq 'del(.sections, .screenshots)')
```
```diff
--- /dev/fd/63  2024-10-07 05:48:20.895568307 +0000
+++ /dev/fd/62  2024-10-07 05:48:20.895568307 +0000
@@ -11,16 +11,16 @@
   "compatibility": [],
   "rating": 98,
   "ratings": {
-    "5": 1143,
-    "4": 21,
-    "3": 8,
+    "1": 15,
     "2": 4,
-    "1": 15
+    "3": 8,
+    "4": 21,
+    "5": 1143
   },
   "num_ratings": 1191,
   "support_threads": 14,
   "support_threads_resolved": 6,
-  "downloaded": 67406328,
+  "downloaded": 67346876,
   "last_updated": "2024-09-27 9:53pm GMT",
   "added": "2017-10-24",
   "homepage": "https://wordpress.org/plugins/classic-editor/",
@@ -64,5 +64,6 @@
     "desrosj": "https://profiles.wordpress.org/desrosj/",
     "luciano-croce": "https://profiles.wordpress.org/luciano-croce/",
     "ironprogrammer": "https://profiles.wordpress.org/ironprogrammer/"
-  }
+  },
+  "download_link_mirror": "https://downloads.mycloudflareproxy_domain.com/classic-editor.1.6.5.zip"
 }
```

Here you see some cosmetic differences for ratings and download counts due to different times the plugin's JSON metadata was captured. Again the locally mirrored and cached copy also adds `download_link_mirror` link to the mirrored WordPress plugin's zip download file.

### 14. **Cloudflare D1 SQLite Database Support**:

With Cloudflare we can also optionally create Cloudflare Workers that are binded to their [Cloudflare D1 SQLite databases](https://developers.cloudflare.com/d1/) as outlined [here](https://developers.cloudflare.com/workers/runtime-apis/bindings/). Cloudflare D1 SQLite database also supports automatic backups and recovery via [Time Travel](https://developers.cloudflare.com/d1/reference/time-travel/). You can track your [Cloudflare D1 SQLite dashboard metrics](#cloudflare-d1-sqlite-dashboard-metrics) as well.

As of October 10, 2024 my POC implementation can be used without binding to a Cloudflare D1 SQLite database for my use case. However, if was to create a web site for mirrored WordPress plugin listing directory via Cloudflare Workers or Cloudflare Pages, I would need to be able to query the locally cached and saved WordPress plugin JSON metadata info and plugin JSON checksum info. 

I was curious how that would implemented so created a third Cloudflare Worker `https://mycloudflare-d1-worker.domain.com` that is binded to existing Cloudflare R2 S3 buckets for `WP_PLUGIN_INFO` and `WP_PLUGIN_STORE` and binded to a Cloudflare D1 SQLite database `DB` variable. This would be a totally separate process from already outlined above system and can operate independently as an optional feature.

Created a shell script `scan_plugins_update_d1.sh` which talks with the Cloudflare Worker `https://mycloudflare-d1-worker.domain.com` which supports several modes for - `single`, `batch`, and `all` so that I can insert a single WordPress by slug name into Cloudflare D1 SQLite database I created or batch or process all WordPress plugins in Cloudflare R2 buckets. The script also supports parallel threads mode for faster insertions into the database. On an libvirtd created KVM VPS with 16 CPU threads (AMD Ryzen 5950X), single threaded runs with 200 batch insertions for 60K WordPress plugins took approximately 120 minutes (2hrs) to complete. With 8 parallel threads and 200 batch insertions per thread, the same 60K plugins took 25 minutes to complete.

An example of checking number WordPress plugins listed in Cloudflare R2 bucket so I can verify with eventually how many plugins are added to Cloudflare D1 SQLite database. Because Cloudflare R2 bucket and D1 SQLite databases are on the same provider, there are performance and speed advantages too.

```bash
time ./scan_plugins_update_d1.sh -u https://mycloudflare-d1-worker.domain.com -C -d
Total number of plugins: 60150

real    0m44.850s
user    0m0.020s
sys     0m0.010s
```

An example is adding to Cloudflare D1 SQLite database `DB` the `akismet` WordPress plugin's JSON metadata and plugin JSON checksum info via the saved info in Cloudflare R2 buckets `WP_PLUGIN_INFO` and `WP_PLUGIN_STORE`:

```bash
./scan_plugins_update_d1.sh -u https://mycloudflare-d1-worker.domain.com -m single -p akismet -d
Making request to: https://mycloudflare-d1-worker.domain.com?mode=single&debug=1&plugin=akismet
Response:
{
  "processed": 1,
  "updated": 1,
  "unchanged": 0,
  "errors": 0,
  "message": "Processing complete",
  "debugInfo": {
    "totalProcessed": 1,
    "updatedInD1": 1,
    "unchanged": 0,
    "errors": 0,
    "errorDetails": []
  }
}
```

The Cloudflare Worker `https://mycloudflare-d1-worker.domain.com` also logs plugin's checksum and message lengths and SQL query length to make sure it isn't hitting [Cloudflare D1 SQLite limits](https://developers.cloudflare.com/d1/platform/limits/). The messages length contain all data that is to be populated into D1 SQLite database.

```json
    {
      "message": [
        "Checksum data size: 5351 bytes"
      ],
      "level": "log",
      "timestamp": 1728653251204
    },
    {
      "message": [
        "Info data size: 14099 bytes"
      ],
      "level": "log",
      "timestamp": 1728653251204
    },
    {
      "message": [
        "SQL query length: 19610 bytes"
      ],
      "level": "log",
      "timestamp": 1728653251667
    }
```

Querying the Cloudflare D1 SQLite database via Cloudflare D1 dashboard.
```bash
> SELECT * FROM plugins WHERE slug = 'akismet';
```

![Cloudflare D1 SQLite database query](/screenshots/cloudflare-d1-sqlite-plugins-database-01.png)

Cloudflare D1 SQLite metrics for test run.

![Cloudflare D1 SQLite database query](/screenshots/cloudflare-d1-sqlite-plugins-database-02.png)

Other D1 SQLite queries with full 60k WordPress plugins added.

![Cloudflare D1 SQLite Dashboard console queries](/screenshots/cf-d1-dashboard-console-01.png)

![Cloudflare D1 SQLite Dashboard console queries](/screenshots/cf-d1-dashboard-console-02.png)

#### Integration WordPress Plugins Download + R2 S3 Store + D1 SQLite Database

Updated `get_plugins_r2.sh` to add Cloudflare D1 SQLite integration so that after downloading and/or populating Cloudflare R2 S3 object storage buckets with JSON meta data and checksum JSON data, the script also calls `scan_plugins_update_d1.sh` to insert plugin's JSON meta data and checksum JSON data into the database via `-i -w https://mycloudflare-d1-worker.domain.com` arguments. These are optional arguments so you can choose whether or not you want to have a Cloudflare D1 SQLite database instance.

Run with 1 thread, debug mode, cache-only mode, forced flag with import update to D1 SQLite database via the Cloudflare D1 SQLite worker url.

```bash
./get_plugins_r2.sh -p 1 -d -c -f -i -w https://mycloudflare-d1-worker.domain.com

Processing plugin: advanced-custom-fields
[DEBUG] Checking latest version and download link for advanced-custom-fields
[DEBUG] Latest version for advanced-custom-fields: 6.3.6.1
[DEBUG] API download link for advanced-custom-fields: https://downloads.wordpress.org/plugin/advanced-custom-fields.6.3.6.1.zip
[DEBUG] Stored version for advanced-custom-fields: 6.3.6
[DEBUG] API-provided download link for advanced-custom-fields: https://downloads.wordpress.org/plugin/advanced-custom-fields.6.3.6.1.zip
[DEBUG] Constructed download link for advanced-custom-fields: https://downloads.wordpress.org/plugin/advanced-custom-fields.6.3.6.1.zip
[DEBUG] Using API-provided download link for advanced-custom-fields: https://downloads.wordpress.org/plugin/advanced-custom-fields.6.3.6.1.zip
[DEBUG] Downloading advanced-custom-fields version 6.3.6.1 through Cloudflare Worker
[DEBUG] Plugin advanced-custom-fields version 6.3.6.1 cached successfully.
Successfully processed advanced-custom-fields.
Time taken for advanced-custom-fields: 0.1124 seconds
[DEBUG] Saving plugin json metadata for advanced-custom-fields version 6.3.6.1
[DEBUG] Plugin metadata for advanced-custom-fields version 6.3.6.1 cached successfully.
[DEBUG] Successfully saved json metadata for advanced-custom-fields.
[DEBUG] Fetching and saving checksums for advanced-custom-fields version 6.3.6.1
[DEBUG] Sending request to Worker for checksums: ?plugin=advanced-custom-fields&version=6.3.6.1&type=checksums&force_update=true&cache_only=true
[DEBUG] Plugin checksums for advanced-custom-fields version 6.3.6.1 cached successfully.
[DEBUG] Successfully fetched and saved checksums for advanced-custom-fields.
[DEBUG] Importing advanced-custom-fields version 6.3.6.1 to D1 database
[DEBUG] D1 import response: Processing plugin: advanced-custom-fields
Making request to: https://mycloudflare-d1-worker.domain.com?mode=single&debug=1&plugin=advanced-custom-fields&batchSize=1
Response:
{
  "processed": 1,
  "updated": 1,
  "unchanged": 0,
  "errors": 0,
  "message": "Processing complete",
  "debugInfo": {
    "totalProcessed": 1,
    "updatedInD1": 1,
    "unchanged": 0,
    "errors": 0,
    "errorDetails": []
  }
}
[DEBUG] Successfully imported advanced-custom-fields to D1 database.
```

<a name="acf"></a>
Yes `advanced-custom-fields` plugin needs updating to it's new download link as Wordpress.org has now hijacked and stolen WPEngine's Advanced Custom Fields (ACF) plugin and renamed it as Secure Custom Fields but kept the plugin slugname and removed all links to ACF Pro upsell.

The stolen hijacked ACF listing on `wordpress.org` now listed name as `Secure Custom Fields` with `6.3.6.2` version:

```json
curl -s https://api.wordpress.org/plugins/info/1.0/advanced-custom-fields.json | jq -r '[.name, .slug, .version, .download_link, .last_updated]'
[
  "Secure Custom Fields",
  "advanced-custom-fields",
  "6.3.6.2",
  "https://downloads.wordpress.org/plugin/advanced-custom-fields.6.3.6.2.zip",
  "2024-10-12 5:47pm GMT"
]
```

My current local mirrored ACF `6.3.6.1` version listing which needs updating:

```json
curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/advanced-custom-fields.json | jq -r '[.name, .slug, .version, .download_link, .last_updated]'
[
  "Advanced Custom Fields (ACF)",
  "advanced-custom-fields",
  "6.3.6.1",
  "https://downloads.wordpress.org/plugin/advanced-custom-fields.6.3.6.1.zip",
  "2024-10-07 5:58pm GMT"
]
```

My local mirror also keeps a backup history of WordPress plugin's JSON metadata just in case though haven't made use of it other than for backup purposes right now. Eventually my plugin mirror system will support mirroring for plugins outside of `wordpress.org`.

![Local mirror plugin JSON metadata backups](/screenshots/wp-plugin-json-metadata-backup-01.png)

## Examples

This example shows the WordPress plugins chosen to be downloaded were retrieved from existing Cloudflare R2 S3 object storage bucket instead of from WordPress.org as they were previously downloaded and cached. If you run script with `-c` cache only mode, it allows checking and updating the cache and R2 bucket without downloading files, useful for preemptive caching and system checks where you do not require actual downloading of plugin zip files to the server you ran the script from. You can jump straight to a demo example of how the mirrored plugin zip files can be installed [here](#demo-wordpress-plugin-install-using-local-mirror).

### Basic Usage

```bash
time ./get_plugins_r2.sh -p 1 -d
```

### Options

- `-d`: Enable debug mode for more detailed logging
- `-p N`: Set the number of parallel download jobs (e.g., `-p 4` for 4 parallel jobs)
- `-a`: Download all available WordPress plugins
- `-l`: Create a list of all plugins without downloading. Saved to file `${WORDPRESS_WORKDIR}/wp-plugin-svn-list.txt`
- `-D y`: Enable download delays
- `-t N`: Set the delay duration in seconds (e.g., `-t 5` for a 5-second delay)
- `-f`: Force update of JSON metadata for all processed plugins
- `-c`: Run in cache-only mode (check and update cache without downloading files)
- `-s N`: Start processing plugins from line N in the SVN list (only used with `-a`)
- `-e N`: End processing plugins at line N in the SVN list (only used with `-a`)

Examples:

```bash
# Run with debug mode
./get_plugins_r2.sh -d

# Run with 4 parallel jobs
./get_plugins_r2.sh -p 4

# Download all plugins instead of selectively predefined plugins
./get_plugins_r2.sh -a

# List all plugins without downloading
./get_plugins_r2.sh -l

# Run with a 5-second delay between downloads
./get_plugins_r2.sh -D y -t 5

# Run with debug mode, 4 parallel jobs, and a 3-second delay
./get_plugins_r2.sh -d -p 4 -D y -t 3

# Force update JSON metadata for all processed plugins
./get_plugins_r2.sh -f

# Run with debug mode, 4 parallel jobs, a 3-second delay, and force update
./get_plugins_r2.sh -d -p 4 -D y -t 3 -f

# Run in cache-only mode
./get_plugins_r2.sh -c

# Run in cache-only mode with debug and force update
./get_plugins_r2.sh -c -d -f

# Process plugins from line 1 to 1000 in the SVN list
./get_plugins_r2.sh -a -s 1 -e 1000

# Process plugins from line 1001 to 2000 in the SVN list
./get_plugins_r2.sh -a -s 1001 -e 2000

# Process plugins from line 2001 to 3000 in the SVN list
./get_plugins_r2.sh -a -s 2001 -e 3000
```

Full `get_plugins_r2.sh` run.

```bash
time ./get_plugins_r2.sh -p 1 -d

Processing plugin: advanced-custom-fields
[DEBUG] Checking latest version and download link for advanced-custom-fields
[DEBUG] Latest version for advanced-custom-fields: 6.3.6
[DEBUG] API download link for advanced-custom-fields: https://downloads.wordpress.org/plugin/advanced-custom-fields.6.3.6.zip
[DEBUG] Stored version for advanced-custom-fields: 6.3.6
[DEBUG] API-provided download link for advanced-custom-fields: https://downloads.wordpress.org/plugin/advanced-custom-fields.6.3.6.zip
[DEBUG] Constructed download link for advanced-custom-fields: https://downloads.wordpress.org/plugin/advanced-custom-fields.6.3.6.zip
[DEBUG] Using API-provided download link for advanced-custom-fields: https://downloads.wordpress.org/plugin/advanced-custom-fields.6.3.6.zip
[DEBUG] Downloading advanced-custom-fields version 6.3.6 through Cloudflare Worker
[DEBUG] Successfully downloaded advanced-custom-fields version 6.3.6 from R2 storage
Successfully processed advanced-custom-fields.
Time taken for advanced-custom-fields: 0.8319 seconds
[DEBUG] Saving plugin json metadata for advanced-custom-fields version 6.3.6
[DEBUG] json metadata for advanced-custom-fields version 6.3.6 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for advanced-custom-fields.
[DEBUG] Fetching and saving checksums for advanced-custom-fields version 6.3.6
[DEBUG] Sending request to Worker for checksums: https://your-worker-url.workers.dev?plugin=advanced-custom-fields&version=6.3.6&type=checksums
[DEBUG] Received response with status code: 200
[DEBUG] Response source: 
[DEBUG] Checksums for advanced-custom-fields version 6.3.6 saved (checksums file already exists)
[DEBUG] Successfully fetched and saved checksums for advanced-custom-fields.
Processing plugin: akismet
[DEBUG] Checking latest version and download link for akismet
[DEBUG] Latest version for akismet: 5.3.3
[DEBUG] API download link for akismet: https://downloads.wordpress.org/plugin/akismet.5.3.3.zip
[DEBUG] Stored version for akismet: 5.3.3
[DEBUG] API-provided download link for akismet: https://downloads.wordpress.org/plugin/akismet.5.3.3.zip
[DEBUG] Constructed download link for akismet: https://downloads.wordpress.org/plugin/akismet.5.3.3.zip
[DEBUG] Using API-provided download link for akismet: https://downloads.wordpress.org/plugin/akismet.5.3.3.zip
[DEBUG] Downloading akismet version 5.3.3 through Cloudflare Worker
[DEBUG] Successfully downloaded akismet version 5.3.3 from R2 storage
Successfully processed akismet.
Time taken for akismet: 0.3040 seconds
[DEBUG] Saving plugin json metadata for akismet version 5.3.3
[DEBUG] json metadata for akismet version 5.3.3 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for akismet.
[DEBUG] Fetching and saving checksums for akismet version 5.3.3
[DEBUG] Sending request to Worker for checksums: https://your-worker-url.workers.dev?plugin=akismet&version=5.3.3&type=checksums
[DEBUG] Received response with status code: 200
[DEBUG] Response source: 
[DEBUG] Checksums for akismet version 5.3.3 saved (checksums file already exists)
[DEBUG] Successfully fetched and saved checksums for akismet.
Processing plugin: amr-cron-manager
[DEBUG] Checking latest version and download link for amr-cron-manager
[DEBUG] Latest version for amr-cron-manager: 2.3
[DEBUG] API download link for amr-cron-manager: https://downloads.wordpress.org/plugin/amr-cron-manager.2.3.zip
[DEBUG] Stored version for amr-cron-manager: 2.3
[DEBUG] API-provided download link for amr-cron-manager: https://downloads.wordpress.org/plugin/amr-cron-manager.2.3.zip
[DEBUG] Constructed download link for amr-cron-manager: https://downloads.wordpress.org/plugin/amr-cron-manager.2.3.zip
[DEBUG] Using API-provided download link for amr-cron-manager: https://downloads.wordpress.org/plugin/amr-cron-manager.2.3.zip
[DEBUG] Downloading amr-cron-manager version 2.3 through Cloudflare Worker
[DEBUG] Successfully downloaded amr-cron-manager version 2.3 from R2 storage
Successfully processed amr-cron-manager.
Time taken for amr-cron-manager: 0.5523 seconds
[DEBUG] Saving plugin json metadata for amr-cron-manager version 2.3
[DEBUG] json metadata for amr-cron-manager version 2.3 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for amr-cron-manager.
[DEBUG] Fetching and saving checksums for amr-cron-manager version 2.3
[DEBUG] Sending request to Worker for checksums: https://your-worker-url.workers.dev?plugin=amr-cron-manager&version=2.3&type=checksums
[DEBUG] Received response with status code: 200
[DEBUG] Response source: 
[DEBUG] Checksums for amr-cron-manager version 2.3 saved (checksums file already exists)
[DEBUG] Successfully fetched and saved checksums for amr-cron-manager.
Processing plugin: assets-manager
[DEBUG] Checking latest version and download link for assets-manager
[DEBUG] Latest version for assets-manager: 1.0.2
[DEBUG] API download link for assets-manager: https://downloads.wordpress.org/plugin/assets-manager.zip
[DEBUG] Stored version for assets-manager: 1.0.2
[DEBUG] API-provided download link for assets-manager: https://downloads.wordpress.org/plugin/assets-manager.zip
[DEBUG] Constructed download link for assets-manager: https://downloads.wordpress.org/plugin/assets-manager.1.0.2.zip
[DEBUG] Using API-provided download link for assets-manager: https://downloads.wordpress.org/plugin/assets-manager.zip
[DEBUG] Downloading assets-manager version 1.0.2 through Cloudflare Worker
[DEBUG] Successfully downloaded assets-manager version 1.0.2 from R2 storage
Successfully processed assets-manager.
Time taken for assets-manager: 0.4126 seconds
[DEBUG] Saving plugin json metadata for assets-manager version 1.0.2
[DEBUG] json metadata for assets-manager version 1.0.2 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for assets-manager.
[DEBUG] Fetching and saving checksums for assets-manager version 1.0.2
[DEBUG] Sending request to Worker for checksums: https://your-worker-url.workers.dev?plugin=assets-manager&version=1.0.2&type=checksums
[DEBUG] Received response with status code: 200
[DEBUG] Response source: 
[DEBUG] Checksums for assets-manager version 1.0.2 saved (checksums file already exists)
[DEBUG] Successfully fetched and saved checksums for assets-manager.
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
Time taken for autoptimize: 0.2655 seconds
[DEBUG] Saving plugin json metadata for autoptimize version 3.1.12
[DEBUG] json metadata for autoptimize version 3.1.12 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for autoptimize.
[DEBUG] Fetching and saving checksums for autoptimize version 3.1.12
[DEBUG] Sending request to Worker for checksums: https://your-worker-url.workers.dev?plugin=autoptimize&version=3.1.12&type=checksums
[DEBUG] Received response with status code: 200
[DEBUG] Response source: 
[DEBUG] Checksums for autoptimize version 3.1.12 saved (checksums file already exists)
[DEBUG] Successfully fetched and saved checksums for autoptimize.
Processing plugin: better-search-replace
[DEBUG] Checking latest version and download link for better-search-replace
[DEBUG] Latest version for better-search-replace: 1.4.7
[DEBUG] API download link for better-search-replace: https://downloads.wordpress.org/plugin/better-search-replace.zip
[DEBUG] Stored version for better-search-replace: 1.4.7
[DEBUG] API-provided download link for better-search-replace: https://downloads.wordpress.org/plugin/better-search-replace.zip
[DEBUG] Constructed download link for better-search-replace: https://downloads.wordpress.org/plugin/better-search-replace.1.4.7.zip
[DEBUG] Using API-provided download link for better-search-replace: https://downloads.wordpress.org/plugin/better-search-replace.zip
[DEBUG] Downloading better-search-replace version 1.4.7 through Cloudflare Worker
[DEBUG] Successfully downloaded better-search-replace version 1.4.7 from R2 storage
Successfully processed better-search-replace.
Time taken for better-search-replace: 0.3155 seconds
[DEBUG] Saving plugin json metadata for better-search-replace version 1.4.7
[DEBUG] json metadata for better-search-replace version 1.4.7 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for better-search-replace.
[DEBUG] Fetching and saving checksums for better-search-replace version 1.4.7
[DEBUG] Sending request to Worker for checksums: https://your-worker-url.workers.dev?plugin=better-search-replace&version=1.4.7&type=checksums
[DEBUG] Received response with status code: 200
[DEBUG] Response source: 
[DEBUG] Checksums for better-search-replace version 1.4.7 saved (checksums file already exists)
[DEBUG] Successfully fetched and saved checksums for better-search-replace.
Processing plugin: cache-enabler
[DEBUG] Checking latest version and download link for cache-enabler
[DEBUG] Latest version for cache-enabler: 1.8.15
[DEBUG] API download link for cache-enabler: https://downloads.wordpress.org/plugin/cache-enabler.1.8.15.zip
[DEBUG] Stored version for cache-enabler: 1.8.15
[DEBUG] API-provided download link for cache-enabler: https://downloads.wordpress.org/plugin/cache-enabler.1.8.15.zip
[DEBUG] Constructed download link for cache-enabler: https://downloads.wordpress.org/plugin/cache-enabler.1.8.15.zip
[DEBUG] Using API-provided download link for cache-enabler: https://downloads.wordpress.org/plugin/cache-enabler.1.8.15.zip
[DEBUG] Downloading cache-enabler version 1.8.15 through Cloudflare Worker
[DEBUG] Successfully downloaded cache-enabler version 1.8.15 from R2 storage
Successfully processed cache-enabler.
Time taken for cache-enabler: 0.5615 seconds
[DEBUG] Saving plugin json metadata for cache-enabler version 1.8.15
[DEBUG] json metadata for cache-enabler version 1.8.15 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for cache-enabler.
[DEBUG] Fetching and saving checksums for cache-enabler version 1.8.15
[DEBUG] Sending request to Worker for checksums: https://your-worker-url.workers.dev?plugin=cache-enabler&version=1.8.15&type=checksums
[DEBUG] Received response with status code: 200
[DEBUG] Response source: 
[DEBUG] Checksums for cache-enabler version 1.8.15 saved (checksums file already exists)
[DEBUG] Successfully fetched and saved checksums for cache-enabler.
Processing plugin: classic-editor
[DEBUG] Checking latest version and download link for classic-editor
[DEBUG] Latest version for classic-editor: 1.6.5
[DEBUG] API download link for classic-editor: https://downloads.wordpress.org/plugin/classic-editor.1.6.5.zip
[DEBUG] Stored version for classic-editor: 1.6.5
[DEBUG] API-provided download link for classic-editor: https://downloads.wordpress.org/plugin/classic-editor.1.6.5.zip
[DEBUG] Constructed download link for classic-editor: https://downloads.wordpress.org/plugin/classic-editor.1.6.5.zip
[DEBUG] Using API-provided download link for classic-editor: https://downloads.wordpress.org/plugin/classic-editor.1.6.5.zip
[DEBUG] Downloading classic-editor version 1.6.5 through Cloudflare Worker
[DEBUG] Successfully downloaded classic-editor version 1.6.5 from R2 storage
Successfully processed classic-editor.
Time taken for classic-editor: 0.3234 seconds
[DEBUG] Saving plugin json metadata for classic-editor version 1.6.5
[DEBUG] json metadata for classic-editor version 1.6.5 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for classic-editor.
[DEBUG] Fetching and saving checksums for classic-editor version 1.6.5
[DEBUG] Sending request to Worker for checksums: https://your-worker-url.workers.dev?plugin=classic-editor&version=1.6.5&type=checksums
[DEBUG] Received response with status code: 200
[DEBUG] Response source: 
[DEBUG] Checksums for classic-editor version 1.6.5 saved (checksums file already exists)
[DEBUG] Successfully fetched and saved checksums for classic-editor.
Plugin download process completed.

real    0m23.510s
user    0m0.700s
sys     0m0.324s
```

```bash
ls -lahrt /home/nginx/domains/plugins.domain.com/public/
total 6.6M
drwxr-sr-x 4 root nginx   39 Sep 30 12:56 ..
-rw-r--r-- 1 root nginx 6.0M Oct  4 03:17 advanced-custom-fields.6.3.6.zip
-rw-r--r-- 1 root nginx 104K Oct  4 03:17 akismet.5.3.3.zip
-rw-r--r-- 1 root nginx 9.3K Oct  4 03:18 amr-cron-manager.2.3.zip
-rw-r--r-- 1 root nginx  20K Oct  4 03:18 assets-manager.1.0.2.zip
-rw-r--r-- 1 root nginx 259K Oct  4 03:18 autoptimize.3.1.12.zip
-rw-r--r-- 1 root nginx 155K Oct  4 03:18 better-search-replace.1.4.7.zip
-rw-r--r-- 1 root nginx  45K Oct  4 03:18 cache-enabler.1.8.15.zip
-rw-r--r-- 1 root nginx  20K Oct  4 03:18 classic-editor.1.6.5.zip
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

### Mirrored Plugin Checksums

The system now includes functionality to fetch, store, and verify plugin checksums. This new feature enhances security by allowing users to verify the integrity of downloaded plugin files against the official WordPress.org checksums. You can jump straight to a demo example of how the mirrored plugin checksum JSON files can be used [here](#demo-wordpress-installed-plugin-checksum-verification).

#### How it works:

1. The system fetches official checksums from WordPress.org for each plugin.
2. Checksums are stored in the R2 bucket alongside plugin files and metadata.
3. Users can verify downloaded plugins against these checksums to ensure file integrity.

Example run for just `autoptimize` plugin:

```
rm -rf /home/wordpress-svn/last_changed_revisions.txt && rm -rf /home/nginx/domains/plugins.domain.com/public/*

time ./get_plugins_r2.sh -p 1 -d

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
Time taken for autoptimize: 0.1435 seconds
[DEBUG] Saving plugin json metadata for autoptimize version 3.1.12
[DEBUG] json metadata for autoptimize version 3.1.12 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for autoptimize.
[DEBUG] Fetching and saving checksums for autoptimize version 3.1.12
[DEBUG] Sending request to Worker for checksums: https://your-worker-url.workers.dev?plugin=autoptimize&version=3.1.12&type=checksums&force_update=true
[DEBUG] Received response with status code: 200
[DEBUG] Response source: 
[DEBUG] Checksums for autoptimize version 3.1.12 saved (checksums file already exists)
[DEBUG] Successfully fetched and saved checksums for autoptimize.
Plugin download process completed.

real    0m1.978s
user    0m0.097s
sys     0m0.045s
```

WordPress plugin download

```
curl -I https://downloads.mycloudflareproxy_domain.com/autoptimize.3.1.12.zip
HTTP/2 200 
date: Fri, 04 Oct 2024 01:56:31 GMT
content-type: application/zip
content-length: 264379
etag: "49dbcac863d2ec3e3ff4675064a943ec"
last-modified: Tue, 01 Oct 2024 16:06:26 GMT
vary: Accept-Encoding
cf-cache-status: HIT
age: 72381
expires: Mon, 04 Nov 2024 01:56:31 GMT
cache-control: public, max-age=2678400
accept-ranges: bytes
server: cloudflare
cf-ray: 8cd18e4f0f897d58-LAX
```

WordPress plugin checksums queried from local mirrored and Cloudflare cached R2 S3 object store version at `https://downloads.mycloudflareproxy_domain.com/plugin-checksums/autoptimize/3.1.12.json` which is meant to replicate the Wordpress.org version at `https://downloads.wordpress.org/plugin-checksums/autoptimize/3.1.2.json`. Added `zip_mirror` field link to local mirror copy of plugin download link too.

```
curl -s https://downloads.mycloudflareproxy_domain.com/plugin-checksums/autoptimize/3.1.12.json | jq -r '[.zip, .zip_mirror]'
[
  "https://downloads.wordpress.org/plugins/autoptimize.3.1.12.zip",
  "https://downloads.mycloudflareproxy_domain.com/autoptimize.3.1.12.zip"
]
```

Full WordPress plugin checksums output. The `zip_mirror` field is ordered at bottom of output. Could reorder it for visual display but not needed.

```
curl -s https://downloads.mycloudflareproxy_domain.com/plugin-checksums/autoptimize/3.1.12.json | jq -r
{
  "plugin": "autoptimize",
  "version": "3.1.12",
  "source": "https://plugins.svn.wordpress.org/autoptimize/tags/3.1.12/",
  "zip": "https://downloads.wordpress.org/plugins/autoptimize.3.1.12.zip",
  "files": {
    "LICENSE": {
      "md5": "8264535c0c4e9c6c335635c4026a8022",
      "sha256": "a45d0bb572ed792ed34627a72621834b3ba92aab6e2cc4e04301dee7a728d753"
    },
    "autoptimize.php": {
      "md5": "6e2e19ab82b51403c53752dc7644dc72",
      "sha256": "a08620d3ff01f11e5b487db915a38c9aa09b6d0c2fad9db30893bf62598013bf"
    },
    "autoptimize_helper.php_example": {
      "md5": "205d3afe9b9f8936b390c48dbfb11bab",
      "sha256": "c74b4a74fd693919e1f3afddcc5d3fa0b1522d5934bffaee2fbf60ccfd39dc8d"
    },
    "classes/autoptimizeBase.php": {
      "md5": "244ccb3b3864a5097974958a7206988a",
      "sha256": "517c79450d349e69d1e2d0918605855d4ed330b02dfec2eb3dc472e20fa10684"
    },
    "classes/autoptimizeCLI.php": {
      "md5": "b944f23eb3d0505dfc4501eecb430ba8",
      "sha256": "bfc6752846fadf2b80db3bba6734f2c5f09c6af24fc2660c196ab2ee57330801"
    },
    "classes/autoptimizeCSSmin.php": {
      "md5": "94ecf9cc6b56b0777a9585abb24e4459",
      "sha256": "158493813a94360b25b0b080815d5fa74974ee79c77ea623128f5b21c6e8e16d"
    },
    "classes/autoptimizeCache.php": {
      "md5": "d27f75afefb66f13913228857c75da1a",
      "sha256": "666fabdc23a15982d2047d3a1c01a14012905af2839f63ebeca0e06adbaadf25"
    },
    "classes/autoptimizeCacheChecker.php": {
      "md5": "d373eccbc28843d7b584e6939a9851ee",
      "sha256": "9428a93e65b6b1686c3c9390d4169d0e8d0911e4a89ef299ed86a844722a5b91"
    },
    "classes/autoptimizeCompatibility.php": {
      "md5": "d374d632ab8f31198c2b1dee5eb88c5a",
      "sha256": "0a1aaaa15324cbd8aa8228b585227099142bed6fb12001a6511292113c1d6938"
    },
    "classes/autoptimizeConfig.php": {
      "md5": "ea64f11180ccdd9c080ee3abca7fc4e6",
      "sha256": "06f15af7d3044748e21ced12b83278d9f7e12b05b767c9dabf045aaed612ea24"
    },
    "classes/autoptimizeCriticalCSSBase.php": {
      "md5": "bff4d80b271006c32120200c20efda3f",
      "sha256": "8a667398e8296d890b6a7f55c908349b6b4de0c388c3fdfbdd9cb449f2557b9b"
    },
    "classes/autoptimizeCriticalCSSCore.php": {
      "md5": "56c4ea2d53baaaa04756228fe34baa64",
      "sha256": "350e5fd3ee001edac63660a915d953b19840f9c11a3ba29b37f2f7a8230c8411"
    },
    "classes/autoptimizeCriticalCSSCron.php": {
      "md5": "43246f6d586b1227cfd89683457b9287",
      "sha256": "8b7f7ebc5ec894fd878f0c28d61795da85df595653a6015656a44e8e471f977a"
    },
    "classes/autoptimizeCriticalCSSEnqueue.php": {
      "md5": "933e49f5fee25511403f2a000d699160",
      "sha256": "cc2a9466ab19446e778fc0c26d09ba7cf27b814c70095628f9b50fc9800e866a"
    },
    "classes/autoptimizeCriticalCSSSettings.php": {
      "md5": "8a7c4edd54a4429e464ad4ac443e8854",
      "sha256": "69b4320a7f2f1b2f037c158da07f1937e6eef066f959d81baddb303a5cc3f3c4"
    },
    "classes/autoptimizeCriticalCSSSettingsAjax.php": {
      "md5": "24fe31304f39dfc39256e567544cd26b",
      "sha256": "a15640d7713934e0159fc1a2a6ea8dfadff5e856ca3c84450b00c673dda24ba2"
    },
    "classes/autoptimizeExitSurvey.php": {
      "md5": "4ea39d1515cfd872686859d18d41c16d",
      "sha256": "3374ab856f52454d8321e02ab4b52630579fff4cbf183071d5d86552f3ac54c3"
    },
    "classes/autoptimizeExtra.php": {
      "md5": "41a26b05794390aa329c7de4215a7852",
      "sha256": "e1a9d6e02545c037f231ea184f0905fa792b60dc1c8ab55fc29821d4a11a26a8"
    },
    "classes/autoptimizeHTML.php": {
      "md5": "5bdf7ff2306d1233bb543f2f1d611979",
      "sha256": "134406ba8659761c53266724f4ede1f9c3ac9392ece93de740cd52c734d6b0f8"
    },
    "classes/autoptimizeImages.php": {
      "md5": "5cd15041edeeabdf1ff3f798f6203a90",
      "sha256": "f78351110bfad5694244e5cee2a95b6fcb9c08fee999b8db97db4e9e0cb1f637"
    },
    "classes/autoptimizeMain.php": {
      "md5": "f24831cfba1c706bd360f73d1bf33a70",
      "sha256": "d91149c20d607c277eaeb157d816b075cc1ead523b46d8db7f09627303c5a713"
    },
    "classes/autoptimizeMetabox.php": {
      "md5": "07cfc8da3c46f19018ff293e5e6d3447",
      "sha256": "4b9ab76596e47c8cc5685f896ea3f39297b3d345e696b170e12ea535aaf9229f"
    },
    "classes/autoptimizeOptionWrapper.php": {
      "md5": "76c8b1abc9e98dff698ded12f3ad6deb",
      "sha256": "c682cd04e98b5b51434786b89da5cad30b05250fc6a098c38bc5230fb2ff0878"
    },
    "classes/autoptimizePartners.php": {
      "md5": "73a94c24630e6cbeb9821d3083f57adb",
      "sha256": "4d03a6a22f4afd8741ab1821154412e6118769307d0b0dd9fbaba83ef1d882bf"
    },
    "classes/autoptimizeProTab.php": {
      "md5": "5ac6c624ed4cf7f924fb00918734184d",
      "sha256": "5a8fdad738be1b3b0c3d67b0d982c85630f7de69d8436ec73488709322bb49eb"
    },
    "classes/autoptimizeScripts.php": {
      "md5": "963d5c657448274aecdd86d2cb4b42d7",
      "sha256": "c53d392cf82a25b9e7c082649a91d263bf6d16d88b50c81321714dd273255353"
    },
    "classes/autoptimizeSpeedupper.php": {
      "md5": "63ae28bacd6791c67b7b07dceea41ccd",
      "sha256": "1b6b8cf39a0769bd75d74904513f02c2ae17393ceff73a7aa1b658f36630c678"
    },
    "classes/autoptimizeStyles.php": {
      "md5": "1f890f572c7a028f877b5c3b08ac921d",
      "sha256": "19910db7577acd9a0d815e865c847ec8baf383f4db577b6eec6221e849ec71e8"
    },
    "classes/autoptimizeToolbar.php": {
      "md5": "2617929eec3c4fb2552bbba8fe53bde3",
      "sha256": "7b0738c7c5130a48f294c6d4a3f07b27c0f98dcdee2b2ee0e1d9eaef1245d6dc"
    },
    "classes/autoptimizeUtils.php": {
      "md5": "cc6426ed2fc0cdb0342c1b39f537f2b9",
      "sha256": "8f3796d9cc141ec24d30afed0504b1d693d7ef500b8a1384bff1901dd7e53fbf"
    },
    "classes/autoptimizeVersionUpdatesHandler.php": {
      "md5": "79ff7f82bd634a6c47e7351a4ed0e2de",
      "sha256": "6f50b62c8ab80c5b7f334546b56b855b38337f78c0cb60d8e15d022b6bd7a82e"
    },
    "classes/critcss-inc/admin_settings_adv.php": {
      "md5": "623a61484ad66df65240b555fc0e1e2b",
      "sha256": "842cc36aa8b8291df9d93c32855fbf37e3eebaadf59b442336967bdfa75180bf"
    },
    "classes/critcss-inc/admin_settings_debug.php": {
      "md5": "549a0ed67fa85bad9cdcd93d7824635b",
      "sha256": "80c2050c00bd092c52eb627b587b12d1028e89a3bc2090976970853951ccc476"
    },
    "classes/critcss-inc/admin_settings_explain.php": {
      "md5": "ecf18d604e3e9208e26c38b2f968ff2e",
      "sha256": "3168f2d19f60999a7d8a4480c0b0e91b1a24102fec41629737f2fd527fe9e112"
    },
    "classes/critcss-inc/admin_settings_impexp.js.php": {
      "md5": "163ea78cc7b54a24656f5388807c2f26",
      "sha256": "b4092386ce39987b23bb515bab11d0b803a1a71bb6d57d38934041ad6a290d91"
    },
    "classes/critcss-inc/admin_settings_key.php": {
      "md5": "9f58a54f1c6f44a5c3dc887747308d6a",
      "sha256": "6f45c21de863b8d247223ea96ae64474d725a20fef4697eb3256cba7700ffbe5"
    },
    "classes/critcss-inc/admin_settings_queue.js.php": {
      "md5": "2bedd84c761996c60f5f7de5c2d3e09e",
      "sha256": "7dce89bf433c98a5ff1eb2470894559ed74a0c0673d4dc77ee7434c7d09a64a9"
    },
    "classes/critcss-inc/admin_settings_queue.php": {
      "md5": "242c6d576fa055136e30ccbf0cd915aa",
      "sha256": "6e863c244e456dda7d7eb94f76df1e3138ce7e043de817f6582d8530fb31d0c8"
    },
    "classes/critcss-inc/admin_settings_rules.js.php": {
      "md5": "41bcea7946cd6d02773fb714c6478544",
      "sha256": "8aa89b1b957fdd10f96a914ed4ba6e574877def154d94069656e3a003146f291"
    },
    "classes/critcss-inc/admin_settings_rules.php": {
      "md5": "9bfdc55c7620611fe667b0ed5454604f",
      "sha256": "937339a6e65433353c21b7343847ce5fd2e5f9470c10f199ecafd9889c637a3d"
    },
    "classes/critcss-inc/css/admin_styles.css": {
      "md5": "8baa259e42a4e1d2513a2ff99ad1cf74",
      "sha256": "9555bdfdd6571689c7256cadf8308c759f10e996380ccb35321251793a5b50a6"
    },
    "classes/critcss-inc/css/ao-tablesorter/asc.gif": {
      "md5": "05d3db0081998106ce3c56735da19c53",
      "sha256": "2986d63c92fe0530ed38c9a575b8a57f30f6e644d133d63a3f7910e7399f9cfd"
    },
    "classes/critcss-inc/css/ao-tablesorter/bg.gif": {
      "md5": "2ee8a6953adc895fbab33a66fa77a0bd",
      "sha256": "13b831eab467260b766212f1d71fb31bef4d67df23aca2c444f24788150675bf"
    },
    "classes/critcss-inc/css/ao-tablesorter/desc.gif": {
      "md5": "b0c55062a15066f60d615abf6ec98746",
      "sha256": "79ce004c07caa11bd126340afd57ad2104e774a1dbfe9763c0dce67592c2ffeb"
    },
    "classes/critcss-inc/css/ao-tablesorter/style.css": {
      "md5": "669034b0d2dcf76e9d5ac66146a7c925",
      "sha256": "7adf54de7dc630ec661e3530bc3541ff46e7a6b9ddd0c3d8aa108fcff96259df"
    },
    "classes/critcss-inc/js/admin_settings.js": {
      "md5": "a5179c478cef924f134f11abb19c282c",
      "sha256": "ca58cc54be7d607b4c3fa117589cdda5e3171bb79452dbb5a19f91371bed1071"
    },
    "classes/critcss-inc/js/jquery.tablesorter.min.js": {
      "md5": "f1cc6ebdef9231e4747473f82d7f2c7c",
      "sha256": "54bb04a582b2bc4f49575ea153acd8c473509a93fd7bc6ef33a019b15fdf4dad"
    },
    "classes/critcss-inc/js/md5.min.js": {
      "md5": "b24893215933dafef9a250b4a46a602d",
      "sha256": "27d221be42096f476245524ecaef8d76d838d5189b16417c79a03ad23763b41f"
    },
    "classes/external/do_not_donate_smallest.png": {
      "md5": "586ba18f1e8e08f25b8117f4db4a0899",
      "sha256": "31317f4ab51b098a726d7c37fe4135d4f9380ea343547bfcb8a30e6022ed031f"
    },
    "classes/external/index.html": {
      "md5": "0ec66da07221a6c5f68fc62571a1b7f7",
      "sha256": "c9a69e377eea7262984d88d3294f64266a2444726aaa87284f5c13c4613a8f2e"
    },
    "classes/external/js/index.html": {
      "md5": "0ec66da07221a6c5f68fc62571a1b7f7",
      "sha256": "c9a69e377eea7262984d88d3294f64266a2444726aaa87284f5c13c4613a8f2e"
    },
    "classes/external/js/jquery.cookie.js": {
      "md5": "f371d4e8cbe6fe960f9e88e755d75ebb",
      "sha256": "b16b491962294905e3cff4ac14f45c7da629b5918d5cd77f54d4b97e93316c8f"
    },
    "classes/external/js/jquery.cookie.min.js": {
      "md5": "c0d03ada6aec64f3c178da25331f0a8f",
      "sha256": "1cc91511f853cb46fbbfaf1bd9c3c55944fbce7f5b0e02e82906502736c30671"
    },
    "classes/external/js/lazysizes.min.js": {
      "md5": "d1edbffbde50cd32ab770746b4140906",
      "sha256": "c4fada4accfa24704b54248bc5ce84acac50b6a059828b7714fe3006786c80c1"
    },
    "classes/external/js/unslider-dots.css": {
      "md5": "3fc3024b132f6a7b81fe535ffffdc5a6",
      "sha256": "92b758fa6195848b306a834a4654683aff3f7b747cf5a65c824677e481cd137d"
    },
    "classes/external/js/unslider.css": {
      "md5": "f5c7a42a618f4f6f7e6ea0677b781e46",
      "sha256": "e8b6bf8321af1ccd3253e3d6d5d0b65288a9b42ab64fd4ba1bad618eafe96bb3"
    },
    "classes/external/js/unslider.min.js": {
      "md5": "41d6943422aa5dcecad652df78f260a7",
      "sha256": "dfe3b7611edf3fbcb1fc8feea9479b017d2c3645596e9e299f6e987beb7c4a18"
    },
    "classes/external/php/ao-minify-html.php": {
      "md5": "a323680c39d2acd5448d8b03fe282226",
      "sha256": "d368f65ff455d07f1a0a283bd0ef8deac31602971305ecd2e1f39aa62a2a4a9e"
    },
    "classes/external/php/index.html": {
      "md5": "0ec66da07221a6c5f68fc62571a1b7f7",
      "sha256": "c9a69e377eea7262984d88d3294f64266a2444726aaa87284f5c13c4613a8f2e"
    },
    "classes/external/php/jsmin.php": {
      "md5": "5a15a0d9e574ec8c816c4ffc2cc8ad87",
      "sha256": "b5ecade31f92cd19f40e535cd173152910e92a2c5575aedb0904e59f366873c1"
    },
    "classes/external/php/persist-admin-notices-dismissal/README.md": {
      "md5": "3a8d0d06b4b8861d3a73a3bac5fe0402",
      "sha256": "6a84c07b52cfcb5ac646828f66c67dd062b4100308465437f10ced85b5c388c3"
    },
    "classes/external/php/persist-admin-notices-dismissal/dismiss-notice.js": {
      "md5": "64012407f5f93d41a8d2b5d44783e78e",
      "sha256": "e7d8333730ec157d11313255f3f86e2cd0f13906a857e32eee81866b2edbf563"
    },
    "classes/external/php/persist-admin-notices-dismissal/persist-admin-notices-dismissal.php": {
      "md5": "f96be65837b77495538fd0f7d8e506cf",
      "sha256": "3d651fff90eba15f5a0ecec752b31307a3cf45d1e8c995572b75fb4f312e3e97"
    },
    "classes/external/php/yui-php-cssmin-bundled/Colors.php": {
      "md5": "96f5364446d60a49a07578c6007212f8",
      "sha256": "8241f3aa9fbff6f93a3965cdc57ad234c6eabc4b8ade1cc4ee07b5a4f12121c4"
    },
    "classes/external/php/yui-php-cssmin-bundled/Minifier.php": {
      "md5": "e36c75393edd3c13ea0e6c2a8f735a3f",
      "sha256": "99f7cca9ba9a7c8b9d188bc35d989d373df7b1a37533d1dacdee860f5f1d4bda"
    },
    "classes/external/php/yui-php-cssmin-bundled/Utils.php": {
      "md5": "a1991bbd4a186ac386c17b2c8d773eff",
      "sha256": "80aca0be7aff734bc0a084da7b3d4c16d1ecd4f8fcfa24d0cb9e0f26b5a16016"
    },
    "classes/external/php/yui-php-cssmin-bundled/index.html": {
      "md5": "0ec66da07221a6c5f68fc62571a1b7f7",
      "sha256": "c9a69e377eea7262984d88d3294f64266a2444726aaa87284f5c13c4613a8f2e"
    },
    "classes/index.html": {
      "md5": "114b8f8a1ef61b647770e5157ed8ce16",
      "sha256": "cebf5190fbd16eb950d39686d4d91adfbe013af57ed359e59d971b875594d999"
    },
    "classes/static/exit-survey/exit-survey.css": {
      "md5": "142e9ea50bbb5355f333d6bf6b6430e8",
      "sha256": "8fab6a78c55fafa7bc01125facad56a2c110f626e6c2ae1cfcb8f0549a02dbea"
    },
    "classes/static/exit-survey/exit-survey.js": {
      "md5": "af4d56930f6d00aeaa5f0d47db6a4c47",
      "sha256": "59da3221da1aed73a8fde62914e3646a04d0f1195920467e97873d760ae602dc"
    },
    "classes/static/loading.gif": {
      "md5": "3fe525943d7c65e42cf4eb3b4b8faadc",
      "sha256": "a32e6e0f11c0e330c93528577ec3fb05a2b0295555bbdb301a1161e1e5ce0194"
    },
    "classes/static/toolbar.css": {
      "md5": "46026fe300d8b28370edbaee60d6f3ef",
      "sha256": "83ccb86844b037ab212b780aec3cfd5b99c8e891684ec26ff33608667fd13e5f"
    },
    "classes/static/toolbar.js": {
      "md5": "75853c230a5698015327421bc68e854c",
      "sha256": "5b914081fa55efa384fdbd44cd5b30ab9af9b3f7c84c323e07526ce1d4199ff8"
    },
    "classes/static/toolbar.min.css": {
      "md5": "f0c26e43e07537fe7aeec160c03fdf55",
      "sha256": "52f87d70523ad1b71837e4f90cf80ee72b3c4abc43ebc7b3005176248f349ed4"
    },
    "classes/static/toolbar.min.js": {
      "md5": "3487659efc81b19d3d385eeb222cf41d",
      "sha256": "8e82bfa65188ed2aaa4971e036b13aa1167f760695971389346037a587ae6a19"
    },
    "config/autoptimize_404_handler.php": {
      "md5": "36c7e6bcf213a60c907be1192ca8363d",
      "sha256": "088676e4bf93e3a4b56b1c0d2e40b4606a81d5d9ff8e78c1e01a0c42e509b09f"
    },
    "config/default.php": {
      "md5": "a7bf33b0b73347530d0ef0320e7fdad8",
      "sha256": "4c228dc78903b21b4497d88d8fe9639b7fd5127716285a84d0c15b456d4d7cf2"
    },
    "config/index.html": {
      "md5": "114b8f8a1ef61b647770e5157ed8ce16",
      "sha256": "cebf5190fbd16eb950d39686d4d91adfbe013af57ed359e59d971b875594d999"
    },
    "index.html": {
      "md5": "114b8f8a1ef61b647770e5157ed8ce16",
      "sha256": "cebf5190fbd16eb950d39686d4d91adfbe013af57ed359e59d971b875594d999"
    },
    "readme.txt": {
      "md5": "f3e1ee970629b4930c97cb78846e49a4",
      "sha256": "36470540e90472dd37b0bfab07012899493863fa6402a7922ae1a42ef2158f1e"
    }
  },
  "zip_mirror": "https://downloads.mycloudflareproxy_domain.com/autoptimize.3.1.12.zip"
}
```

### Cached Plugin

Example of Cloudflare CDN cached plugin compared to Wordpress plugin download. Using Cloudflare CDN caching will improve download speed and improve latency for Wordpress plugin downloads and for your web site in general. See my tutorial write up on Cloudflare community forum - [Improving Time To First Byte (TTFB) With Cloudflare](https://community.cloudflare.com/t/improving-time-to-first-byte-ttfb-with-cloudflare/390367/). You can also speed how fast Cloudflare CDN cached mirrored WordPress themes zip files are [here](#wordpress-themes-cloudflare-cdn-benchmarks).

Cloudflare CDN cached

```
curl -I https://downloads.mycloudflareproxy_domain.com/autoptimize.3.1.12.zip
HTTP/2 200 
date: Thu, 03 Oct 2024 05:50:14 GMT
content-type: application/zip
content-length: 264379
etag: "49dbcac863d2ec3e3ff4675064a943ec"
last-modified: Tue, 01 Oct 2024 16:06:26 GMT
vary: Accept-Encoding
cf-cache-status: HIT
age: 4
expires: Sun, 03 Nov 2024 05:50:14 GMT
cache-control: public, max-age=2678400
accept-ranges: bytes
server: cloudflare
cf-ray: 8ccaa74b58c37cc7-LAX
```

Original Wordpress plugin

```
curl -I https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
HTTP/1.1 200 OK
Server: nginx
Date: Thu, 03 Oct 2024 05:48:34 GMT
Content-Type: application/octet-stream
Content-Length: 264379
Connection: close
Content-Disposition: attachment; filename=autoptimize.3.1.12.zip
Last-Modified: Thu, 25 Jul 2024 17:15:10 GMT
Accept-Ranges: bytes
Access-Control-Allow-Methods: GET, HEAD
Access-Control-Allow-Origin: *
Alt-Svc: h3=":443"; ma=86400
X-nc: MISS ord 7
```

<a name="curltimes"></a>
Comparing [`curltimes.sh`](https://github.com/centminmod/curltimes) for both

| Metric | Run | Cloudflare CDN (s) | Original WordPress (s) | Difference (s) | Percentage Difference (%) |
|--------|-----|---------------------|------------------------|----------------|---------------------------|
| DNS Lookup | 1 | 0.017216 | 0.012032 | -0.005184 | -43.09% |
|            | 2 | 0.017077 | 0.012507 | -0.004570 | -36.55% |
|            | 3 | 0.017610 | 0.012564 | -0.005046 | -40.17% |
| **DNS Avg** |  | **0.017301** | **0.012368** | **-0.004933** | **-39.89%** |
| Connect | 1 | 0.018738 | 0.065434 | 0.046696 | 71.39% |
|         | 2 | 0.019041 | 0.066386 | 0.047345 | 71.31% |
|         | 3 | 0.019858 | 0.066524 | 0.046666 | 70.16% |
| **Connect Avg** |  | **0.019212** | **0.066115** | **0.046903** | **71.00%** |
| SSL | 1 | 0.043311 | 0.187870 | 0.144559 | 76.92% |
|     | 2 | 0.042507 | 0.189180 | 0.146673 | 77.53% |
|     | 3 | 0.042925 | 0.189554 | 0.146629 | 77.34% |
| **SSL Avg** |  | **0.042914** | **0.188868** | **0.145954** | **77.27%** |
| TTFB | 1 | 0.100405 | 0.416238 | 0.315833 | 75.86% |
|      | 2 | 0.096419 | 0.248653 | 0.152234 | 61.21% |
|      | 3 | 0.099871 | 0.364501 | 0.264630 | 72.60% |
| **TTFB Avg** |  | **0.098898** | **0.343131** | **0.244233** | **71.18%** |
| Total Time | 1 | 0.107244 | 0.665148 | 0.557904 | 83.88% |
|            | 2 | 0.103766 | 0.528787 | 0.425021 | 80.38% |
|            | 3 | 0.108014 | 0.630060 | 0.522046 | 82.86% |
| **Total Avg** |  | **0.106341** | **0.607998** | **0.501657** | **82.50%** |

#### Notes:
- Times are in seconds (s)
- Lower times indicate faster performance
- Cloudflare CDN is consistently faster in most metrics, except for initial DNS lookup
- Average total time improvement: 0.501657 seconds (approximately 82.5% faster)

#### Interpretation:
1. **DNS Lookup**: Cloudflare is slightly slower (by ~0.005s or -39.89%), likely due to the additional Cloudflare DNS resolution.
2. **Connect**: Cloudflare is significantly faster (by ~0.047s or 71.00%), possibly due to closer server proximity.
3. **SSL**: Cloudflare performs much better (by ~0.146s or 77.27%), likely due to optimized SSL handshake.
4. **Time to First Byte (TTFB)**: Cloudflare is substantially faster (by ~0.244s or 71.18%), indicating quicker server response.
5. **Total Time**: Cloudflare CDN delivers the file much faster (by ~0.502s or 82.50%), which is a significant improvement.

Cloudflare CDN cached

```
./curltimes.sh json https://downloads.mycloudflareproxy_domain.com/autoptimize.3.1.12.zip
curl 7.76.1
TLSv1.2 ECDHE-ECDSA-CHACHA20-POLY1305
Connected to downloads.mycloudflareproxy_domain.com (104.xxx.xxx.xxx) port 443 (#0)
Cloudflare proxied https://downloads.mycloudflareproxy_domain.com/autoptimize.3.1.12.zip
Sample Size: 3

DNS,Connect,SSL,Wait,TTFB,Total Time
{
        "time_dns":             0.017216,
        "time_connect":         0.018738,
        "time_appconnect":      0.043311,
        "time_pretransfer":     0.043366,
        "time_ttfb":            0.100405,
        "time_total":           0.107244
}{
        "time_dns":             0.017077,
        "time_connect":         0.019041,
        "time_appconnect":      0.042507,
        "time_pretransfer":     0.042568,
        "time_ttfb":            0.096419,
        "time_total":           0.103766
}{
        "time_dns":             0.017610,
        "time_connect":         0.019858,
        "time_appconnect":      0.042925,
        "time_pretransfer":     0.042988,
        "time_ttfb":            0.099871,
        "time_total":           0.108014
}

```

Original Wordpress plugin

```
./curltimes.sh json https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
TLSv1.2 ECDHE-ECDSA-CHACHA20-POLY1305
Connected to downloads.wordpress.org (198.143.164.250) port 443 (#0)
Sample Size: 3

DNS,Connect,SSL,Wait,TTFB,Total Time
{
        "time_dns":             0.012032,
        "time_connect":         0.065434,
        "time_appconnect":      0.187870,
        "time_pretransfer":     0.187953,
        "time_ttfb":            0.416238,
        "time_total":           0.665148
}{
        "time_dns":             0.012507,
        "time_connect":         0.066386,
        "time_appconnect":      0.189180,
        "time_pretransfer":     0.189303,
        "time_ttfb":            0.248653,
        "time_total":           0.528787
}{
        "time_dns":             0.012564,
        "time_connect":         0.066524,
        "time_appconnect":      0.189554,
        "time_pretransfer":     0.189640,
        "time_ttfb":            0.364501,
        "time_total":           0.630060
}
```

### wget download speed test

| Source | Run | Download Speed (MB/s) |
|--------|-----|------------------------|
| Cloudflare CDN | 1 | 37.8 |
| Cloudflare CDN | 2 | 46.8 |
| WordPress.org | 1 | 0.9 |
| WordPress.org | 2 | 1.05 |

<details>
<summary>Show raw wget WordPress plugin zip download benchmarks</summary>

Cloudflare CDN cached

Run 1 = 37.8MB/s

```
wget -O /dev/null https://downloads.mycloudflareproxy_domain.com/autoptimize.3.1.12.zip
--2024-10-03 10:46:44--  https://downloads.mycloudflareproxy_domain.com/autoptimize.3.1.12.zip
Resolving downloads.mycloudflareproxy_domain.com (downloads.mycloudflareproxy_domain.com)... 104.xxx.xxx.xxx, 104.xxx.xxx.xxx, 2606:xxxx, ...
Connecting to downloads.mycloudflareproxy_domain.com (downloads.mycloudflareproxy_domain.com)|104.xxx.xxx.xxx|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 264379 (258K) [application/zip]
Saving to: ‘/dev/null’

/dev/null                                      100%[==================================================================================================>] 258.18K  --.-KB/s    in 0.007s  

2024-10-03 10:46:44 (37.8 MB/s) - ‘/dev/null’ saved [264379/264379]
```

Run 2 = 46.8MB/s

```
wget -O /dev/null https://downloads.mycloudflareproxy_domain.com/autoptimize.3.1.12.zip
--2024-10-03 10:47:09--  https://downloads.mycloudflareproxy_domain.com/autoptimize.3.1.12.zip
Resolving downloads.mycloudflareproxy_domain.com (downloads.mycloudflareproxy_domain.com)... 104.xxx.xxx.xxx, 104.xxx.xxx.xxx, 2606:xxyy, ...
Connecting to downloads.mycloudflareproxy_domain.com (downloads.mycloudflareproxy_domain.com)|104.xxx.xxx.xxx|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 264379 (258K) [application/zip]
Saving to: ‘/dev/null’

/dev/null                                      100%[==================================================================================================>] 258.18K  --.-KB/s    in 0.005s  

2024-10-03 10:47:09 (46.8 MB/s) - ‘/dev/null’ saved [264379/264379]
```

Original Wordpress plugin

Run 1 = 900KB/s

```
wget -O /dev/null https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
--2024-10-03 10:46:30--  https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
Resolving downloads.wordpress.org (downloads.wordpress.org)... 198.143.164.250
Connecting to downloads.wordpress.org (downloads.wordpress.org)|198.143.164.250|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 264379 (258K) [application/octet-stream]
Saving to: ‘/dev/null’

/dev/null                                      100%[==================================================================================================>] 258.18K   900KB/s    in 0.3s    

2024-10-03 10:46:30 (900 KB/s) - ‘/dev/null’ saved [264379/264379]
```

Run 2 = 1.05MB/s

```
wget -O /dev/null https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
--2024-10-03 10:47:05--  https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
Resolving downloads.wordpress.org (downloads.wordpress.org)... 198.143.164.250
Connecting to downloads.wordpress.org (downloads.wordpress.org)|198.143.164.250|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 264379 (258K) [application/octet-stream]
Saving to: ‘/dev/null’

/dev/null                                      100%[==================================================================================================>] 258.18K  1.05MB/s    in 0.2s    

2024-10-03 10:47:05 (1.05 MB/s) - ‘/dev/null’ saved [264379/264379
```
</details>

Looking closer at Wordpress.org download, not even sure they are properly caching the plugin files they are serving?  Looks like caching per datacenter as last test was a HIT - though no difference in download speed. Did 4 additional runs inspecting HTTP response headers and I see, `EXPIRED`, `MISS`, `MISS`, `HIT` statuses respectively for `x-nc` header.

```
wget -S -O /dev/null https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
--2024-10-03 11:10:17--  https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
Resolving downloads.wordpress.org (downloads.wordpress.org)... 198.143.164.250
Connecting to downloads.wordpress.org (downloads.wordpress.org)|198.143.164.250|:443... connected.
HTTP request sent, awaiting response... 
  HTTP/1.1 200 OK
  Server: nginx
  Date: Thu, 03 Oct 2024 11:10:18 GMT
  Content-Type: application/octet-stream
  Content-Length: 264379
  Connection: close
  Content-Disposition: attachment; filename=autoptimize.3.1.12.zip
  Last-Modified: Thu, 25 Jul 2024 17:15:10 GMT
  Access-Control-Allow-Methods: GET, HEAD
  Access-Control-Allow-Origin: *
  Alt-Svc: h3=":443"; ma=86400
  X-nc: EXPIRED ord 7
  Accept-Ranges: bytes
Length: 264379 (258K) [application/octet-stream]
Saving to: ‘/dev/null’

/dev/null                                      100%[==================================================================================================>] 258.18K   875KB/s    in 0.3s    

2024-10-03 11:10:18 (875 KB/s) - ‘/dev/null’ saved [264379/264379]
```
```
wget -S -O /dev/null https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
--2024-10-03 11:10:21--  https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
Resolving downloads.wordpress.org (downloads.wordpress.org)... 198.143.164.250
Connecting to downloads.wordpress.org (downloads.wordpress.org)|198.143.164.250|:443... connected.
HTTP request sent, awaiting response... 
  HTTP/1.1 200 OK
  Server: nginx
  Date: Thu, 03 Oct 2024 11:10:21 GMT
  Content-Type: application/octet-stream
  Content-Length: 264379
  Connection: close
  Content-Disposition: attachment; filename=autoptimize.3.1.12.zip
  Last-Modified: Thu, 25 Jul 2024 17:15:10 GMT
  Accept-Ranges: bytes
  Access-Control-Allow-Methods: GET, HEAD
  Access-Control-Allow-Origin: *
  Alt-Svc: h3=":443"; ma=86400
  X-nc: MISS ord 4
Length: 264379 (258K) [application/octet-stream]
Saving to: ‘/dev/null’

/dev/null                                      100%[==================================================================================================>] 258.18K   933KB/s    in 0.3s    

2024-10-03 11:10:22 (933 KB/s) - ‘/dev/null’ saved [264379/264379]
```
```
wget -S -O /dev/null https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
--2024-10-03 11:10:23--  https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
Resolving downloads.wordpress.org (downloads.wordpress.org)... 198.143.164.250
Connecting to downloads.wordpress.org (downloads.wordpress.org)|198.143.164.250|:443... connected.
HTTP request sent, awaiting response... 
  HTTP/1.1 200 OK
  Server: nginx
  Date: Thu, 03 Oct 2024 11:10:24 GMT
  Content-Type: application/octet-stream
  Content-Length: 264379
  Connection: close
  Content-Disposition: attachment; filename=autoptimize.3.1.12.zip
  Last-Modified: Thu, 25 Jul 2024 17:15:10 GMT
  Accept-Ranges: bytes
  Access-Control-Allow-Methods: GET, HEAD
  Access-Control-Allow-Origin: *
  Alt-Svc: h3=":443"; ma=86400
  X-nc: MISS ord 6
Length: 264379 (258K) [application/octet-stream]
Saving to: ‘/dev/null’

/dev/null                                      100%[==================================================================================================>] 258.18K  1.03MB/s    in 0.2s    

2024-10-03 11:10:24 (1.03 MB/s) - ‘/dev/null’ saved [264379/264379]
```
```
wget -S -O /dev/null https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
--2024-10-03 11:13:29--  https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
Resolving downloads.wordpress.org (downloads.wordpress.org)... 198.143.164.250
Connecting to downloads.wordpress.org (downloads.wordpress.org)|198.143.164.250|:443... connected.
HTTP request sent, awaiting response... 
  HTTP/1.1 200 OK
  Server: nginx
  Date: Thu, 03 Oct 2024 11:13:29 GMT
  Content-Type: application/octet-stream
  Content-Length: 264379
  Connection: close
  Content-Disposition: attachment; filename=autoptimize.3.1.12.zip
  Last-Modified: Thu, 25 Jul 2024 17:15:10 GMT
  Access-Control-Allow-Methods: GET, HEAD
  Access-Control-Allow-Origin: *
  Alt-Svc: h3=":443"; ma=86400
  X-nc: HIT ord 7
  Accept-Ranges: bytes
Length: 264379 (258K) [application/octet-stream]
Saving to: ‘/dev/null’

/dev/null                                      100%[==================================================================================================>] 258.18K  1.05MB/s    in 0.2s    

2024-10-03 11:13:29 (1.05 MB/s) - ‘/dev/null’ saved [264379/264379]
```

### Mirrored WordPress Plugin API End Point

Mirrored and locally cached in Cloudflare R2 bucket plugin JSON metadata where `api.mycloudflareproxy_domain.com` is Cloudflare orange cloud proxy enabled domain zone hostname enabled for Cloudflare R2 bucket public access for `WP_PLUGIN_INFO` bucket referenced in Cloudflare Worker that `get_plugins_r2.sh` talks to.

* Local mirrored: `https://api.mycloudflareproxy_domain.com/plugins/info/1.0/autoptimize.json`
* Original Wordpress API end point: `https://api.wordpress.org/plugins/info/1.0/autoptimize.json`

Also modified the saved JSON metadata to insert an additional field for `download_link_mirror` which also lists the mirrored download url for the WordPress plugin along with existing `download_link` download link.

```
curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/autoptimize.json | jq -r '[.download_link, .download_link_mirror]'
[
  "https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip",
  "https://downloads.mycloudflareproxy_domain.com/autoptimize.3.1.12.zip"
]
```

This mirrored and locally cached JSON metadata endpoint `https://api.mycloudflareproxy_domain.com/plugins/info/1.0/` is also optionally used as the source data url with a **WordPress Plugin 1.2 API Bridge Worker** which is created using a separate Cloudflare Worker. It is designed to bridge the gap between the WordPress Plugin API 1.0 and 1.2 versions `https://api.wordpress.org/plugins/info/1.0` vs `https://api.wordpress.org/plugins/info/1.2`. It allows clients to query plugin information using the 1.2 API format while fetching data from either a mirrored 1.0 API endpoint or the official WordPress.org 1.0 API, providing flexibility and reliability in data retrieval

  ```bash
  curl -s -H "Accept: application/json" "https://api.mycloudflareproxy_domain.com/plugins/info/1.2/?action=plugin_information&slug=autoptimize&locale=en_US" | jq -r '[.name, .slug, .version, .download_link, .tested, .requires_php]'
  [
    "Autoptimize",
    "autoptimize",
    "3.1.12",
    "https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip",
    "6.6.2",
    "5.6"
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

### Github Hosted Plugins

With the possibility of some WordPress plugins being hosted on Github repositories instead of `wordpress.org`, I need to update my mirror system to support this. 

This is first version attempt at supporting this via a config file `/home/wordpress-plugins/github_plugins.txt` which lists each Github hosted plugins details for `plugin-slug:github-username/repo-name`. So for the example of WPEngine's Advanced Custom Fields WordPress plugin hosted at https://github.com/AdvancedCustomFields/acf, it would end up something the below example where I am only defining a small sample of WordPress plugins to download and mirror. Note the Github hosted WordPress plugin checksum JSON file doesn't exist write now. Working on handling that next - ideally would be nice if WordPress plugins hosted on Github would have a publicly available plugin JSON metadata at path `/plugins/info/1.0/advanced-custom-fields.json`  and checksum file JSON file `wordpress.org` equivalents path at `/plugin-checksums/advanced-custom-fields/6.3.8.json` as to what `wordpress.org` hosts. So folks like myself can point to the WordPress plugin's Github hosted JSON metadata and checksum file JSON files.

Apparently once you update to WPEngine's latest plugins, future plugin updates come from their servers https://wpengine.com/support/installing-and-updating-free-wp-engine-plugins-and-themes/. 

#### Other WordPress Plugins Not Hosted On Github

As far as I know, Nitropack doesn't have Github hosted url, so haven't catered for it yet. So far the following WordPress plugins are not hosted on :

* Nitropack https://support.nitropack.io/en/articles/9991759-installing-and-upgrading-to-the-latest-version-of-nitropack
* GravityPDF [announced their hosting their WordPress on their own site](https://gravitypdf.com/news/installing-and-upgrading-to-the-canonical-version-of-gravity-pdf/) moving off of `wordpress.org`. They have changed their slug name from `gravity-forms-pdf-extended` to `gravity-pdf`.

It would be nice if WordPress plugins hosted on their own sites still give folks some labeling or indication of the plugin name, slug name, version via JSON metadata file at path `/plugins/info/1.0/advanced-custom-fields.json` and JSON files checksum file `/plugin-checksums/advanced-custom-fields/6.3.8.json` for mirrors to be able to latch on to and identify the plugins and do file checksum verifications ([example](#demo-wordpress-installed-plugin-checksum-verification)). Below Github hosted plugins example was able to happen because I could get some of that information from their respective Github releases tag, name and download links i.e. for ACF https://github.com/AdvancedCustomFields/acf/releases.

Example for GravityPDF:

```bash
curl -s https://api.wordpress.org/plugins/info/1.0/gravity-forms-pdf-extended.json | jq -r '[.name, .slug, .version, .download_link, .last_updated]'
[
  "Gravity PDF",
  "gravity-forms-pdf-extended",
  "6.12.0",
  "https://downloads.wordpress.org/plugin/gravity-forms-pdf-extended.6.12.0.zip",
  "2024-10-15 3:43am GMT"
]
```
```bash
curl -s https://downloads.wordpress.org/plugin-checksums/gravity-forms-pdf-extended/6.12.0.json | jq -r '[.plugin, .version, .source, .zip]'
[
  "gravity-forms-pdf-extended",
  "6.12.0",
  "https://plugins.svn.wordpress.org/gravity-forms-pdf-extended/tags/6.12.0/",
  "https://downloads.wordpress.org/plugins/gravity-forms-pdf-extended.6.12.0.zip"
]
```

Now test run for Github hosted WordPress plugins:

Defined in `get_plugins_r2.sh` specific plugins to test mirror downloading.

```bash
ADDITIONAL_PLUGINS=(
    "advanced-custom-fields"
    "autoptimize"
    "better-search-replace"
    "wp-migrate-db"
    "wp-amazon-s3-and-cloudfront"
    "wp-offload-ses-lite"
    "classic-editor"
)
```

Example run with `-d` debug mode and single threaded `-p 1`:

```bash
touch /home/wordpress-plugins/github_plugins.txt

echo 'advanced-custom-fields:AdvancedCustomFields/acf' >> /home/wordpress-plugins/github_plugins.txt
echo 'wp-migrate-db:deliciousbrains/wp-migrate-db' >> /home/wordpress-plugins/github_plugins.txt
echo 'wp-amazon-s3-and-cloudfront:deliciousbrains/wp-amazon-s3-and-cloudfront' >> /home/wordpress-plugins/github_plugins.txt
echo 'wp-offload-ses-lite:deliciousbrains/wp-offload-ses-lite' >> /home/wordpress-plugins/github_plugins.txt
echo 'better-search-replace:deliciousbrains/better-search-replace' >> /home/wordpress-plugins/github_plugins.txt

time ./get_plugins_r2.sh -p 1 -d

Processing plugin: advanced-custom-fields
[DEBUG] Processing GitHub-hosted plugin: advanced-custom-fields
[DEBUG] Updated JSON metadata for GitHub plugin advanced-custom-fields version 6.3.8 based on WordPress.org data
[DEBUG] Latest version for advanced-custom-fields: 6.3.8
[DEBUG] API download link for advanced-custom-fields: https://github.com/AdvancedCustomFields/acf/releases/download/6.3.8/advanced-custom-fields-6.3.8.zip
[DEBUG] Stored version for advanced-custom-fields: 6.3.8
advanced-custom-fields is up-to-date and exists in mirror directory. Skipping download...
[DEBUG] Saving plugin json metadata for advanced-custom-fields version 6.3.8
[DEBUG] json metadata for advanced-custom-fields version 6.3.8 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for advanced-custom-fields.
[DEBUG] Fetching and saving checksums for advanced-custom-fields version 6.3.8
[DEBUG] Sending request to Worker for checksums: ?plugin=advanced-custom-fields&version=6.3.8&type=checksums
[DEBUG] Received response with status code: 500
Error: Failed to save checksums for advanced-custom-fields version 6.3.8 (HTTP status: 500)
[DEBUG] Error response: Failed to fetch plugin checksums from WordPress
[DEBUG] Failed to fetch and save checksums for advanced-custom-fields.
Processing plugin: autoptimize
[DEBUG] Processing WordPress.org plugin: autoptimize
[DEBUG] Checking latest version and download link for autoptimize
[DEBUG] Latest version for autoptimize: 3.1.12
[DEBUG] API download link for autoptimize: https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip
[DEBUG] Stored version for autoptimize: 3.1.12
autoptimize is up-to-date and exists in mirror directory. Skipping download...
[DEBUG] Saving plugin json metadata for autoptimize version 3.1.12
[DEBUG] json metadata for autoptimize version 3.1.12 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for autoptimize.
[DEBUG] Fetching and saving checksums for autoptimize version 3.1.12
[DEBUG] Sending request to Worker for checksums: ?plugin=autoptimize&version=3.1.12&type=checksums
[DEBUG] Received response with status code: 200
[DEBUG] Response source: 
[DEBUG] Checksums for autoptimize version 3.1.12 saved (checksums file already exists)
[DEBUG] Successfully fetched and saved checksums for autoptimize.
Processing plugin: better-search-replace
[DEBUG] Processing GitHub-hosted plugin: better-search-replace
[DEBUG] Updated JSON metadata for GitHub plugin better-search-replace version 1.4.9 based on WordPress.org data
[DEBUG] Latest version for better-search-replace: 1.4.9
[DEBUG] API download link for better-search-replace: https://github.com/deliciousbrains/better-search-replace/releases/download/1.4.9/better-search-replace.1.4.9.zip
[DEBUG] Stored version for better-search-replace: 1.4.9
better-search-replace is up-to-date and exists in mirror directory. Skipping download...
[DEBUG] Saving plugin json metadata for better-search-replace version 1.4.9
[DEBUG] json metadata for better-search-replace version 1.4.9 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for better-search-replace.
[DEBUG] Fetching and saving checksums for better-search-replace version 1.4.9
[DEBUG] Sending request to Worker for checksums: ?plugin=better-search-replace&version=1.4.9&type=checksums
[DEBUG] Received response with status code: 500
Error: Failed to save checksums for better-search-replace version 1.4.9 (HTTP status: 500)
[DEBUG] Error response: Failed to fetch plugin checksums from WordPress
[DEBUG] Failed to fetch and save checksums for better-search-replace.
Processing plugin: classic-editor
[DEBUG] Processing WordPress.org plugin: classic-editor
[DEBUG] Checking latest version and download link for classic-editor
[DEBUG] Latest version for classic-editor: 1.6.5
[DEBUG] API download link for classic-editor: https://downloads.wordpress.org/plugin/classic-editor.1.6.5.zip
[DEBUG] Stored version for classic-editor: 1.6.5
classic-editor is up-to-date and exists in mirror directory. Skipping download...
[DEBUG] Saving plugin json metadata for classic-editor version 1.6.5
[DEBUG] json metadata for classic-editor version 1.6.5 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for classic-editor.
[DEBUG] Fetching and saving checksums for classic-editor version 1.6.5
[DEBUG] Sending request to Worker for checksums: ?plugin=classic-editor&version=1.6.5&type=checksums
[DEBUG] Received response with status code: 200
[DEBUG] Response source: 
[DEBUG] Checksums for classic-editor version 1.6.5 saved (checksums file already exists)
[DEBUG] Successfully fetched and saved checksums for classic-editor.
Processing plugin: wp-amazon-s3-and-cloudfront
[DEBUG] Processing GitHub-hosted plugin: wp-amazon-s3-and-cloudfront
[DEBUG] Generated basic JSON metadata for GitHub plugin wp-amazon-s3-and-cloudfront version 3.2.9
[DEBUG] Latest version for wp-amazon-s3-and-cloudfront: 3.2.9
[DEBUG] API download link for wp-amazon-s3-and-cloudfront: https://github.com/deliciousbrains/wp-amazon-s3-and-cloudfront/releases/download/3.2.9/amazon-s3-and-cloudfront.3.2.9.zip
[DEBUG] Stored version for wp-amazon-s3-and-cloudfront: 
[DEBUG] API-provided download link for wp-amazon-s3-and-cloudfront: https://github.com/deliciousbrains/wp-amazon-s3-and-cloudfront/releases/download/3.2.9/amazon-s3-and-cloudfront.3.2.9.zip
[DEBUG] Constructed download link for wp-amazon-s3-and-cloudfront: https://downloads.wordpress.org/plugin/wp-amazon-s3-and-cloudfront.3.2.9.zip
[DEBUG] Using API-provided download link for wp-amazon-s3-and-cloudfront: https://github.com/deliciousbrains/wp-amazon-s3-and-cloudfront/releases/download/3.2.9/amazon-s3-and-cloudfront.3.2.9.zip
[DEBUG] Downloading wp-amazon-s3-and-cloudfront version 3.2.9 through Cloudflare Worker
[DEBUG] Skipped download for wp-amazon-s3-and-cloudfront 3.2.9 zip (already exists locally)
Successfully processed wp-amazon-s3-and-cloudfront.
Time taken for wp-amazon-s3-and-cloudfront: 1.1201 seconds
[DEBUG] Saving plugin json metadata for wp-amazon-s3-and-cloudfront version 3.2.9
[DEBUG] json metadata for wp-amazon-s3-and-cloudfront version 3.2.9 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for wp-amazon-s3-and-cloudfront.
[DEBUG] Fetching and saving checksums for wp-amazon-s3-and-cloudfront version 3.2.9
[DEBUG] Sending request to Worker for checksums: ?plugin=wp-amazon-s3-and-cloudfront&version=3.2.9&type=checksums
[DEBUG] Received response with status code: 500
Error: Failed to save checksums for wp-amazon-s3-and-cloudfront version 3.2.9 (HTTP status: 500)
[DEBUG] Error response: Failed to fetch plugin checksums from WordPress
[DEBUG] Failed to fetch and save checksums for wp-amazon-s3-and-cloudfront.
Processing plugin: wp-migrate-db
[DEBUG] Processing GitHub-hosted plugin: wp-migrate-db
[DEBUG] Updated JSON metadata for GitHub plugin wp-migrate-db version 2.7.0 based on WordPress.org data
[DEBUG] Latest version for wp-migrate-db: 2.7.0
[DEBUG] API download link for wp-migrate-db: https://github.com/deliciousbrains/wp-migrate-db/releases/download/2.7.0/wp-migrate-db.2.7.0.zip
[DEBUG] Stored version for wp-migrate-db: 
[DEBUG] API-provided download link for wp-migrate-db: https://github.com/deliciousbrains/wp-migrate-db/releases/download/2.7.0/wp-migrate-db.2.7.0.zip
[DEBUG] Constructed download link for wp-migrate-db: https://downloads.wordpress.org/plugin/wp-migrate-db.2.7.0.zip
[DEBUG] Using API-provided download link for wp-migrate-db: https://github.com/deliciousbrains/wp-migrate-db/releases/download/2.7.0/wp-migrate-db.2.7.0.zip
[DEBUG] Downloading wp-migrate-db version 2.7.0 through Cloudflare Worker
[DEBUG] Skipped download for wp-migrate-db 2.7.0 zip (already exists locally)
Successfully processed wp-migrate-db.
Time taken for wp-migrate-db: 1.1863 seconds
[DEBUG] Saving plugin json metadata for wp-migrate-db version 2.7.0
[DEBUG] json metadata for wp-migrate-db version 2.7.0 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for wp-migrate-db.
[DEBUG] Fetching and saving checksums for wp-migrate-db version 2.7.0
[DEBUG] Sending request to Worker for checksums: ?plugin=wp-migrate-db&version=2.7.0&type=checksums
[DEBUG] Received response with status code: 500
Error: Failed to save checksums for wp-migrate-db version 2.7.0 (HTTP status: 500)
[DEBUG] Error response: Failed to fetch plugin checksums from WordPress
[DEBUG] Failed to fetch and save checksums for wp-migrate-db.
Processing plugin: wp-offload-ses-lite
[DEBUG] Processing GitHub-hosted plugin: wp-offload-ses-lite
[DEBUG] Generated basic JSON metadata for GitHub plugin wp-offload-ses-lite version 1.7.1
[DEBUG] Latest version for wp-offload-ses-lite: 1.7.1
[DEBUG] API download link for wp-offload-ses-lite: https://github.com/deliciousbrains/wp-offload-ses-lite/releases/download/1.7.1/wp-ses.1.7.1.zip
[DEBUG] Stored version for wp-offload-ses-lite: 
[DEBUG] API-provided download link for wp-offload-ses-lite: https://github.com/deliciousbrains/wp-offload-ses-lite/releases/download/1.7.1/wp-ses.1.7.1.zip
[DEBUG] Constructed download link for wp-offload-ses-lite: https://downloads.wordpress.org/plugin/wp-offload-ses-lite.1.7.1.zip
[DEBUG] Using API-provided download link for wp-offload-ses-lite: https://github.com/deliciousbrains/wp-offload-ses-lite/releases/download/1.7.1/wp-ses.1.7.1.zip
[DEBUG] Downloading wp-offload-ses-lite version 1.7.1 through Cloudflare Worker
[DEBUG] Skipped download for wp-offload-ses-lite 1.7.1 zip (already exists locally)
Successfully processed wp-offload-ses-lite.
Time taken for wp-offload-ses-lite: 1.3354 seconds
[DEBUG] Saving plugin json metadata for wp-offload-ses-lite version 1.7.1
[DEBUG] json metadata for wp-offload-ses-lite version 1.7.1 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for wp-offload-ses-lite.
[DEBUG] Fetching and saving checksums for wp-offload-ses-lite version 1.7.1
[DEBUG] Sending request to Worker for checksums: ?plugin=wp-offload-ses-lite&version=1.7.1&type=checksums
[DEBUG] Received response with status code: 500
Error: Failed to save checksums for wp-offload-ses-lite version 1.7.1 (HTTP status: 500)
[DEBUG] Error response: Failed to fetch plugin checksums from WordPress
[DEBUG] Failed to fetch and save checksums for wp-offload-ses-lite.
Plugin download process completed.

real    0m28.686s
user    0m2.005s
sys     0m0.233s
```

Local downloaded WordPress plugin zip files are as follows with the newer ACF and other Github hosted version now downloaded. Notice `better-search-replace` 1.4.7 came from `wordpress.org` while 1.4.9 is from Githb hosted version now.

```bash
ls -lAhrt /home/nginx/domains/plugins.domain.com/public
total 14M
-rw-r--r-- 1 root nginx 6.1M Oct 14 17:54 advanced-custom-fields.6.3.8.zip
-rw-r--r-- 1 root nginx 259K Oct 14 17:54 autoptimize.3.1.12.zip
-rw-r--r-- 1 root nginx 155K Oct 14 17:54 better-search-replace.1.4.7.zip
-rw-r--r-- 1 root nginx  20K Oct 14 17:54 classic-editor.1.6.5.zip
-rw-r--r-- 1 root nginx 162K Oct 15 07:55 better-search-replace.1.4.9.zip
-rw-r--r-- 1 root nginx 3.5M Oct 15 07:58 wp-amazon-s3-and-cloudfront.3.2.9.zip
-rw-r--r-- 1 root nginx 1.6M Oct 15 07:58 wp-migrate-db.2.7.0.zip
-rw-r--r-- 1 root nginx 2.2M Oct 15 07:58 wp-offload-ses-lite.1.7.1.zip
```

Querying the locally mirrored Advanced Custom Fields's and other WordPress plugin's JSON metadata file shows the `download_link` pointing to Github hosted zip file and also `download_link_mirror` hosted zip file for my local mirrored version.

`advanced-custom-fields`

```bash
curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/advanced-custom-fields.json | jq -r '[.download_link, .download_link_mirror]'
[
  "https://github.com/AdvancedCustomFields/acf/releases/download/6.3.8/advanced-custom-fields-6.3.8.zip",
  "https://downloads.mycloudflareproxy_domain.com/advanced-custom-fields.6.3.8.zip"
]
```

The extended version output for locally mirrored Advanced Custom Fields and other plugin's JSON metadata file where:

- `name` updated to Github hosted release name = `Advanced Custom Fields v6.3.8`
- `slug` still the same
- `version` updated to Github hosted release version = `6.3.8`
- `download_link` updated to Github hosted plugin zip file download link
- `download_link_mirror` added my local mirror plugin zip file download link.

`advanced-custom-fields`

```bash
curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/advanced-custom-fields.json | jq -r '[.name, .slug, .version, .download_link, .download_link_mirror]'
[
  "Advanced Custom Fields v6.3.8",
  "advanced-custom-fields",
  "6.3.8",
  "https://github.com/AdvancedCustomFields/acf/releases/download/6.3.8/advanced-custom-fields-6.3.8.zip",
  "https://downloads.mycloudflareproxy_domain.com/advanced-custom-fields.6.3.8.zip"
]
```

`autoptimize`

```bash
curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/autoptimize.json | jq -r '[.name, .slug, .version, .download_link, .download_link_mirror]'
[
  "Autoptimize",
  "autoptimize",
  "3.1.12",
  "https://downloads.wordpress.org/plugin/autoptimize.3.1.12.zip",
  "https://downloads.mycloudflareproxy_domain.com/autoptimize.3.1.12.zip"
]
```

`better-search-replace`

```bash
curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/better-search-replace.json | jq -r '[.name, .slug, .version, .download_link, .download_link_mirror]'
[
  "1.4.9",
  "better-search-replace",
  "1.4.9",
  "https://github.com/deliciousbrains/better-search-replace/releases/download/1.4.9/better-search-replace.1.4.9.zip",
  "https://downloads.mycloudflareproxy_domain.com/better-search-replace.1.4.9.zip"
]
```

`wp-amazon-s3-and-cloudfront`

```bash
curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/wp-amazon-s3-and-cloudfront.json | jq -r '[.name, .slug, .version, .download_link, .download_link_mirror]'
[
  "3.2.9",
  "wp-amazon-s3-and-cloudfront",
  "3.2.9",
  "https://github.com/deliciousbrains/wp-amazon-s3-and-cloudfront/releases/download/3.2.9/amazon-s3-and-cloudfront.3.2.9.zip",
  "https://downloads.mycloudflareproxy_domain.com/wp-amazon-s3-and-cloudfront.3.2.9.zip"
]
```

`wp-migrate-db`

```bash
curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/wp-migrate-db.json | jq -r '[.name, .slug, .version, .download_link, .download_link_mirror]'
[
  "2.7.0",
  "wp-migrate-db",
  "2.7.0",
  "https://github.com/deliciousbrains/wp-migrate-db/releases/download/2.7.0/wp-migrate-db.2.7.0.zip",
  "https://downloads.mycloudflareproxy_domain.com/wp-migrate-db.2.7.0.zip"
]
```

`wp-offload-ses-lite`

```bash
curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/wp-offload-ses-lite.json | jq -r '[.name, .slug, .version, .download_link, .download_link_mirror]'
[
  "1.7.1",
  "wp-offload-ses-lite",
  "1.7.1",
  "https://github.com/deliciousbrains/wp-offload-ses-lite/releases/download/1.7.1/wp-ses.1.7.1.zip",
  "https://downloads.mycloudflareproxy_domain.com/wp-offload-ses-lite.1.7.1.zip"
]
```

HTTP response headers from local mirror plugin's zip file download links that are saved in Cloudflare R2 S3 object storage:

```bash
curl -I https://downloads.mycloudflareproxy_domain.com/advanced-custom-fields.6.3.8.zip
HTTP/2 200 
date: Mon, 14 Oct 2024 23:12:29 GMT
content-type: application/zip
content-length: 6310074
etag: "a713def0f24445cba251da9f58847bff"
last-modified: Mon, 14 Oct 2024 22:54:14 GMT
vary: Accept-Encoding
cf-cache-status: HIT
age: 3
expires: Mon, 14 Oct 2024 23:17:29 GMT
cache-control: public, max-age=300
accept-ranges: bytes
server: cloudflare
cf-ray: 8d2b4127fe436bc8-DFW
alt-svc: h3=":443"; ma=86400
```

```bash
curl -I https://downloads.mycloudflareproxy_domain.com/autoptimize.3.1.12.zip
HTTP/2 404 
date: Tue, 15 Oct 2024 13:05:30 GMT
content-type: text/html
content-length: 27150
vary: Accept-Encoding
cf-cache-status: HIT
age: 1
expires: Tue, 15 Oct 2024 13:10:30 GMT
cache-control: public, max-age=300
server: cloudflare
cf-ray: 8d3005686fac0bac-DFW
alt-svc: h3=":443"; ma=86400
```

```bash
curl -I https://downloads.mycloudflareproxy_domain.com/better-search-replace.1.4.9.zip
HTTP/2 200 
date: Tue, 15 Oct 2024 13:05:57 GMT
content-type: application/zip
content-length: 164921
etag: "d060997bc07adcca4117869b1c56a0b5"
last-modified: Tue, 15 Oct 2024 12:55:16 GMT
vary: Accept-Encoding
cf-cache-status: HIT
age: 4
expires: Tue, 15 Oct 2024 13:10:57 GMT
cache-control: public, max-age=300
accept-ranges: bytes
server: cloudflare
cf-ray: 8d30060b0bfc6b57-DFW
alt-svc: h3=":443"; ma=86400
```

```bash
curl -I https://downloads.mycloudflareproxy_domain.com/wp-amazon-s3-and-cloudfront.3.2.9.zip
HTTP/2 200 
date: Tue, 15 Oct 2024 13:06:25 GMT
content-type: application/zip
content-length: 3599584
etag: "93e33ee77d267364453b32d2a3581ef5"
last-modified: Tue, 15 Oct 2024 12:58:33 GMT
vary: Accept-Encoding
cf-cache-status: HIT
age: 5
expires: Tue, 15 Oct 2024 13:11:25 GMT
cache-control: public, max-age=300
accept-ranges: bytes
server: cloudflare
cf-ray: 8d3006beac262e76-DFW
alt-svc: h3=":443"; ma=86400
```

```bash
curl -I https://downloads.mycloudflareproxy_domain.com/wp-migrate-db.2.7.0.zip
HTTP/2 200 
date: Tue, 15 Oct 2024 13:06:58 GMT
content-type: application/zip
content-length: 1612918
etag: "308587b8b92c656cc065a1c4973cc15a"
last-modified: Tue, 15 Oct 2024 12:58:39 GMT
vary: Accept-Encoding
cf-cache-status: HIT
age: 6
expires: Tue, 15 Oct 2024 13:11:58 GMT
cache-control: public, max-age=300
accept-ranges: bytes
server: cloudflare
cf-ray: 8d30078b2983e7e7-DFW
alt-svc: h3=":443"; ma=86400
```

```bash
curl -I https://downloads.mycloudflareproxy_domain.com/wp-offload-ses-lite.1.7.1.zip
HTTP/2 200 
date: Tue, 15 Oct 2024 13:07:36 GMT
content-type: application/zip
content-length: 2255713
etag: "a227e9deee787719ec4e126ad01b2d5a"
last-modified: Tue, 15 Oct 2024 12:58:46 GMT
vary: Accept-Encoding
cf-cache-status: HIT
age: 5
expires: Tue, 15 Oct 2024 13:12:36 GMT
cache-control: public, max-age=300
accept-ranges: bytes
server: cloudflare
cf-ray: 8d300876f9b0e96a-DFW
alt-svc: h3=":443"; ma=86400
```

### Demo WordPress Plugin Install Using Local Mirror

An demo example of installing classic-editor WordPress plugin from local mirror I setup above.

Here's my demo WordPress install details.

```
/usr/local/nginx/conf/wpincludes/wp.domain.com/wpinfo.sh
WP-CLI 2.11.0
WP-Home    https://wp.domain.com
WP-SiteURL https://wp.domain.com
WordPress  version:   6.6.2   
Database   revision:  57155   
TinyMCE    version:   4.9110  (49110-20201110)
Package    language:  en_US   
+--------+---------------------------+--------------+---------------------------+--------------------------------+---------------+
| ID     | user_login                | display_name | user_email                | user_registered                | roles         |
+--------+---------------------------+--------------+---------------------------+--------------------------------+---------------+
| 270422 | zfTHxZdCjyOI2Wc1ijVyyhFKc | George       | user@domain.com           |                                |               |
+--------+---------------------------+--------------+---------------------------+--------------------------------+---------------+
+----------------------+------------------------------------------------------------------+----------+
| name                 | value                                                            | type     |
+----------------------+------------------------------------------------------------------+----------+
| table_prefix         | 27994_                                                           | variable |
| WP_CACHE             | 1                                                                | constant |
| DB_NAME              | wp27714970db_217913                                              | constant |
| DB_USER              | wpdb2179u289199                                                  | constant |
| DB_PASSWORD          | wpdbguZLh7nyyBfs82rD0tdrAgp31550                                 | constant |
| DB_HOST              | localhost                                                        | constant |
| DB_CHARSET           | utf8                                                             | constant |
| DB_COLLATE           |                                                                  | constant |
| DISABLE_WP_CRON      |                                                                  | constant |
| WP_AUTO_UPDATE_CORE  | minor                                                            | constant |
| WP_POST_REVISIONS    | 10                                                               | constant |
| EMPTY_TRASH_DAYS     | 10                                                               | constant |
| WP_CRON_LOCK_TIMEOUT | 60                                                               | constant |
| CONCATENATE_SCRIPTS  |                                                                  | constant |
| AUTH_KEY             | >EIS=P]/9ZNYZXeP:Rq<57n.}g@g+6<c<dUn*;fM@}k~i!++$:G:5E05U7&*Fimi | constant |
| SECURE_AUTH_KEY      | <5?#Q=.]Zhy)?>}pylPKce8?G4~#P]cO`ar]t+qq%lA6]>Xh1U|z1:6z/wQ,!<uq | constant |
| LOGGED_IN_KEY        | Unbk[.R*lotu}-a;MkXpG>{jfLWiN/4D!~*utpTvD24d!CHXHsh!:Dc(S7}4eYf: | constant |
| NONCE_KEY            | 8<~A#rc3Za,A(K*Ai`V6W+1bM|J98W#mJqV4zuVUsMWXWK-L* 4PQoop0nJ;%]yp | constant |
| AUTH_SALT            | `Y$8(lcu3nqF+:?6=;W{SbQS/=Qz#,-Yw@bbCn,V!U92BnyeVX^Q]X,4M:i[x!c7 | constant |
| SECURE_AUTH_SALT     | bZ$CUXQR.@T7]~H?6ZwZ(bWsTL$-X+=5d??gT^iomZ+U[v&k>Ii$W^aprwH?*0d0 | constant |
| LOGGED_IN_SALT       | t8$akq.p.ph^&roH|7p,aGp7Q%?ap!cM:zA/?FO`Ce-1_aHfoeiZs4Wqi`:#Vrl] | constant |
| NONCE_SALT           | .>-51kySB$p}$yLS^~y9zkz[73g7#ifx3RoVG7,e?iaol<<7}98[]l!SmrD|$s=z | constant |
| WP_CACHE_KEY_SALT    | (>4Q3dQ8F]&VZ>>v.xj-_(j}#<#Th<L1]=ROLg9@{S.5%QKJ>Y_[ S)aS2X4XssI | constant |
| WP_DEBUG             |                                                                  | constant |
+----------------------+------------------------------------------------------------------+----------+
+-------------------------------+----------+--------+---------+----------------+-------------+
| name                          | status   | update | version | update_version | auto_update |
+-------------------------------+----------+--------+---------+----------------+-------------+
| autoptimize                   | active   | none   | 3.1.12  |                | off         |
| disable-xml-rpc               | active   | none   | 1.0.1   |                | off         |
| sucuri-scanner                | active   | none   | 1.9.5   |                | off         |
+-------------------------------+----------+--------+---------+----------------+-------------+
+-------------------+----------+--------+---------+----------------+-------------+
| name              | status   | update | version | update_version | auto_update |
+-------------------+----------+--------+---------+----------------+-------------+
| twentytwentyfour  | active   | none   | 1.2     |                | off         |
| twentytwentythree | inactive | none   | 1.5     |                | off         |
| twentytwentytwo   | inactive | none   | 1.8     |                | off         |
+-------------------+----------+--------+---------+----------------+-------------+
```

Now to install a WordPress plugin from my local mirror hosted on Cloudlare R2 S3 object storage backed by Cloudflare CDN caching. You can query either WordPress or local mirror's JSON metadata to obtain the download url zip file.

* Local mirrored: `https://api.mycloudflareproxy_domain.com/plugins/info/1.0/classic-editor.json`
* Original Wordpress API end point: `https://api.wordpress.org/plugins/info/1.0/classic-editor.json`

As I have modified the saved JSON metadata to insert an additional field for `download_link_mirror` which also lists the mirrored download url for the WordPress plugin along with existing `download_link` download link.

```
curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/classic-editor.json | jq -r '[.download_link, .download_link_mirror]'
[
  "https://downloads.wordpress.org/plugin/classic-editor.1.6.5.zip",
  "https://downloads.mycloudflareproxy_domain.com/classic-editor.1.6.5.zip"
]
```

If you queried WordPress.org JSON metadata, you'll get the official plugin zip download url too.

```
curl -s https://api.wordpress.org/plugins/info/1.0/classic-editor.json | jq -r '[.download_link]'
[
  "https://downloads.wordpress.org/plugin/classic-editor.1.6.5.zip"
]
```

Let's grab my mirror's classic-editor plugin zip file via mirrored JSON metadata file.

```
curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/classic-editor.json | jq -r '.download_link_mirror'

https://downloads.mycloudflareproxy_domain.com/classic-editor.1.6.5.zip
```

Scripted wget download

```
download=$(curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/classic-editor.json | jq -r '.download_link_mirror')

wget -S $download
```
```
wget -S $download
--2024-10-05 22:13:49--  https://downloads.mycloudflareproxy_domain.com/classic-editor.1.6.5.zip
Resolving downloads.mycloudflareproxy_domain.com (downloads.mycloudflareproxy_domain.com)... 104.xx.xxx.xxx, 104.xx.xxx.xxx, 2606:xxx.xxx.xxx, ...
Connecting to downloads.mycloudflareproxy_domain.com (downloads.mycloudflareproxy_domain.com)|104.xx.xxx.xxx|:443... connected.
HTTP request sent, awaiting response... 
  HTTP/1.1 200 OK
  Date: Sat, 05 Oct 2024 22:13:49 GMT
  Content-Type: application/zip
  Content-Length: 19693
  Connection: keep-alive
  ETag: "563ed3f42294d2b34a82eeb0249eb204"
  Last-Modified: Tue, 01 Oct 2024 16:06:32 GMT
  Vary: Accept-Encoding
  CF-Cache-Status: HIT
  Age: 296
  Expires: Tue, 05 Nov 2024 22:13:49 GMT
  Cache-Control: public, max-age=2678400
  Accept-Ranges: bytes
  Server: cloudflare
  CF-RAY: 8ce0c2d9086808da-LAX
Length: 19693 (19K) [application/zip]
Saving to: ‘classic-editor.1.6.5.zip’

classic-editor.1.6.5.zip         100%[========================================================>]  19.23K  --.-KB/s    in 0.001s  

2024-10-05 22:13:49 (23.7 MB/s) - ‘classic-editor.1.6.5.zip’ saved [19693/19693]
```

Now we have 3 ways of installing classic-editor plugin using WP-CLI tool's native plugin install command https://developer.wordpress.org/cli/commands/plugin/install/:

1. Install plugin zip using default wordpress.org download source
2. Installing from local mirror downloaded plugin zip file
3. Installing from remote mirror server's plugin zip file

#### 1. Install using default wordpress.org download source

Notice installing package from `https://downloads.wordpress.org/plugin/classic-editor.1.6.5.zip`.

```
wp plugin install classic-editor
```
```
wp plugin install classic-editor
Installing Classic Editor (1.6.5)
Downloading installation package from https://downloads.wordpress.org/plugin/classic-editor.1.6.5.zip...
Using cached file '/root/.wp-cli/cache/plugin/classic-editor-1.6.5.zip'...
Unpacking the package...
Installing the plugin...
Plugin installed successfully.
Success: Installed 1 of 1 plugins.
```

and active it (yes we can do install and activate in one command too)

```
wp plugin activate classic-editor
Plugin 'classic-editor' activated.
Success: Activated 1 of 1 plugins.
```

#### 2. Installing from local mirror downloaded plugin zip file

```
wp plugin install ./classic-editor.1.6.5.zip
```
```
wp plugin install ./classic-editor.1.6.5.zip
Unpacking the package...
Installing the plugin...
Plugin installed successfully.
Success: Installed 1 of 1 plugins.
```

activate it

```
wp plugin activate classic-editor
Plugin 'classic-editor' activated.
Success: Activated 1 of 1 plugins.
```

#### 3. Installing from remote mirror server's plugin zip file

Notice installing plugin zip from local mirrored copy at `https://downloads.mycloudflareproxy_domain.com/classic-editor.1.6.5.zip`.

```
wp plugin install https://downloads.mycloudflareproxy_domain.com/classic-editor.1.6.5.zip
```
```
wp plugin install https://downloads.mycloudflareproxy_domain.com/classic-editor.1.6.5.zip
Downloading installation package from https://downloads.mycloudflareproxy_domain.com/classic-editor.1.6.5.zip...
Unpacking the package...
Installing the plugin...
Plugin installed successfully.
Success: Installed 1 of 1 plugins.
```

activate it

```
wp plugin activate classic-editor
Plugin 'classic-editor' activated.
Success: Activated 1 of 1 plugins.
```

Checking installed WordPress plugin list

```
wp plugin list
+-------------------------------+----------+--------+---------+----------------+-------------+
| name                          | status   | update | version | update_version | auto_update |
+-------------------------------+----------+--------+---------+----------------+-------------+
| akismet                       | inactive | none   | 5.3.3   |                | off         |
| autoptimize                   | active   | none   | 3.1.12  |                | off         |
| classic-editor                | inactive | none   | 1.6.5   |                | off         |
| disable-xml-rpc               | active   | none   | 1.0.1   |                | off         |
| sucuri-scanner                | active   | none   | 1.9.5   |                | off         |
+-------------------------------+----------+--------+---------+----------------+-------------+
```

For WordPress plugin update notifications, you could write a script that reads your WordPress installations plugin list using WP-CLI tool and use WP-CLI plugin status https://developer.wordpress.org/cli/commands/plugin/status/ to check for updates or script to check via mirrored JSON metadata query and have it trigger a WP-CLI plugin update command https://developer.wordpress.org/cli/commands/plugin/update/. For added notifications, the script could setup mobile/tablet push notifications via servcies like pushover.net to alert you of new updates, sucessfull/failed updates etc.

#### Demo WordPress Installed Plugin Checksum Verification

Haven't gotten around to modifying WP-CLI tool's plugin `verify-checksums` option https://developer.wordpress.org/cli/commands/plugin/verify-checksums/ to use my local mirror endpoint. So for now created a standalone script that can take a few command arguments to pass the `-p` path of your WordPress install and `-u` the remote url to the plugin's checksum JSON file `https://downloads.mycloudflareproxy_domain.com/plugin-checksums/classic-editor/1.6.5.json` which is meant to replicate the Wordpress.org version at `https://downloads.wordpress.org/plugin-checksums/classic-editor/1.6.5.json`

```bash
./wp-plugin-checksums.sh -p /home/nginx/domains/wp.domain.com/public -u https://downloads.mycloudflareproxy_domain.com/plugin-checksums/classic-editor/1.6.5.json
Checksum verification: OK
```

`-d` debug mode lists checksum verification against each file separately

```bash
./wp-plugin-checksums.sh -d -p /home/nginx/domains/wp.domain.com/public -u https://downloads.mycloudflareproxy_domain.com/plugin-checksums/classic-editor/1.6.5.json
[OK] LICENSE.md: Checksum matches
[OK] classic-editor.php: Checksum matches
[OK] js/block-editor-plugin.js: Checksum matches
[OK] readme.txt: Checksum matches
Checksum verification: OK
```
```bash
ls -lAh /home/nginx/domains/wp.domain.com/public/wp-content/plugins/classic-editor/
total 68K
-rw-r--r-- 1 nginx nginx  19K Oct  5 22:27 LICENSE.md
-rw-r--r-- 1 nginx nginx  37K Oct  5 22:27 classic-editor.php
drwxr-xr-x 2 nginx nginx   36 Oct  5 22:27 js
-rw-r--r-- 1 nginx nginx 7.7K Oct  5 22:27 readme.txt
```

Can also take the official WordPress plugin checksum JSON file as source too.

```bash
./wp-plugin-checksums.sh -p /home/nginx/domains/wp.domain.com/public -u https://downloads.wordpress.org/plugin-checksums/classic-editor/1.6.5.json
Checksum verification: OK
```

`-d` debug mode lists checksum verification against each file separately

```bash
./wp-plugin-checksums.sh -d -p /home/nginx/domains/wp.domain.com/public -u https://downloads.wordpress.org/plugin-checksums/classic-editor/1.6.5.json
[OK] LICENSE.md: Checksum matches
[OK] classic-editor.php: Checksum matches
[OK] js/block-editor-plugin.js: Checksum matches
[OK] readme.txt: Checksum matches
Checksum verification: OK
```

`-e` mode allows you to set the API endpoint domain you want to pull the plugins' JSON checksum files for all WordPress plugins installed. In below example `autoptimize-gzip` is a plugin installed outside of `wordpress.org` which enables `autoptimize` plugin's file pre-compression feature for gzip and brotli. So `autoptimize-gzip` would not have an associated checksum JSON file.

From local mirror API endpoint `downloads.mycloudflareproxy_domain.com`. Will only report invalid checksums by default when ran without `-d` debug mode:

```bash
./wp-plugin-checksums.sh -p /home/nginx/domains/wp.domain.com/public -e downloads.mycloudflareproxy_domain.com
No checksum data available for autoptimize-gzip (version 0.1)
Skipping checksum verification for autoptimize-gzip (version 0.1)
Checksum verification: OK
```

Official WordPress API endpoint `downloads.wordpress.org`. Will only report invalid checksums by default when ran without `-d` debug mode:

```bash
./wp-plugin-checksums.sh -p /home/nginx/domains/wp.domain.com/public -e downloads.wordpress.org
Failed to fetch or empty checksum data for autoptimize-gzip (version 0.1)
Skipping checksum verification for autoptimize-gzip (version 0.1)
Checksum verification: OK
```

You can get the plugin's version number via local mirrored plugin API 1.0 or official WordPress API 1.0 endpoints

Local mirror `api.mycloudflareproxy_domain.com`:

```bash
curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/classic-editor.json | jq -r '.version'
1.6.5
```

Official `api.wordpress.org`:

```bash
curl -s https://api.wordpress.org/plugins/info/1.0/classic-editor.json | jq -r '.version'
1.6.5
```

So something like:

```bash
# get plugin version from local mirror plugin JSON metadata
ver=$(curl -s https://api.mycloudflareproxy_domain.com/plugins/info/1.0/classic-editor.json | jq -r '.version')

# query local mirror plugin JSON checksum data
./wp-plugin-checksums.sh -p /home/nginx/domains/wp.domain.com/public -u https://downloads.mycloudflareproxy_domain.com/plugin-checksums/classic-editor/${ver}.json
Checksum verification: OK
```

## Plugin Mirror Screenshots

Using Github Workflow to automate the script.

![Github Workflow](/screenshots/get_plugins_r2_workflow-01.png)

![Github Workflow](/screenshots/get_plugins_r2_workflow-02.png)

Example of Cloudflare R2 S3 buckets populated with all [WordPress plugin zip files](#cached-plugin) and [JSON metadata files](#mirrored-wordpress-plugin-api-end-point) using S3Browser to view the listings.

![Cloudflare R2 S3 Bucket](/screenshots/allplugins-get_plugins_r2_s3browser-listing-plugins-01.png)

![Cloudflare R2 S3 Bucket](/screenshots/allplugins-get_plugins_r2_s3browser-listing-plugins-json-metadata-01.png)

[Plugin checksum API](#mirrored-plugin-checksums) mirrored data i.e. at `https://downloads.mycloudflareproxy_domain.com/plugin-checksums/autoptimize/3.1.12.json` which is meant to replicate the Wordpress.org version at `https://downloads.wordpress.org/plugin-checksums/autoptimize/3.1.2.json`.

![Plugin Checksums in Cloudflare R2 S3 Bucket](/screenshots/allplugins-get_plugins_r2_s3browser-listing-plugins-checksums-01.png)

![Plugin Checksums in Cloudflare R2 S3 Bucket](/screenshots/get_plugins_r2_s3browser-listing-plugins-checksums-02.png)

The count is off and it seems some plugin names are non-English and my script needs to account for this:

```
Error: Invalid plugin slug. for plugin 海阔淘宝相关宝贝插件
Skipping 海阔淘宝相关宝贝插件 due to version fetch error.
```

## WordPress Themes Mirror

The above outlines my proof of concept WordPress plugins mirroring. Next stage is to replicate the same mirror system for WordPress themes' zip and JSON metadata files using `get_themes_r2.sh` shell script which talks with Cloudflare Worker `get_themes_r2.js`.

There are currently 27,494 WordPress themes listed in Wordpress themes SVN repository at https://themes.svn.wordpress.org/ not all are published so need to filter them.

### get_themes_r2.sh Basic Usage

To run the script with default settings:

```bash
./get_themes_r2.sh
```

### get_themes_r2.sh Options

- `-d`: Enable debug mode for more detailed logging
- `-p N`: Set the number of parallel download jobs (e.g., `-p 4` for 4 parallel jobs)
- `-a`: Download all available WordPress themes
- `-l`: Create a list of all themes without downloading. Saved to file `${WORDPRESS_WORKDIR}/wp-theme-svn-list.txt`
- `-D y`: Enable download delays
- `-t N`: Set the delay duration in seconds (e.g., `-t 5` for a 5-second delay)
- `-f`: Force update of JSON metadata for all processed themes
- `-c`: Run in cache-only mode (check and update cache without downloading files)
- `-s N`: Start processing themes from line N in the SVN list (only used with `-a`)
- `-e N`: End processing themes at line N in the SVN list (only used with `-a`)
- `-i`: Enable import to D1 database
- `-w D1_WORKER_URL`: Set the URL for the D1 Worker

Examples:

```bash
# Run with debug mode
./get_themes_r2.sh -d

# Run with 4 parallel jobs
./get_themes_r2.sh -p 4

# Download all themes
./get_themes_r2.sh -a

# List all themes without downloading
./get_themes_r2.sh -l

# Run with a 5-second delay between downloads
./get_themes_r2.sh -D y -t 5

# Run with debug mode, 4 parallel jobs, and a 3-second delay
./get_themes_r2.sh -d -p 4 -D y -t 3

# Force update JSON metadata for all processed themes
./get_themes_r2.sh -f

# Run with debug mode, 4 parallel jobs, a 3-second delay, and force update
./get_themes_r2.sh -d -p 4 -D y -t 3 -f

# Run in cache-only mode
./get_themes_r2.sh -c

# Run in cache-only mode with debug and force update
./get_themes_r2.sh -c -d -f

# Process themes from line 1 to 1000 in the SVN list
./get_themes_r2.sh -a -s 1 -e 1000

# Process themes from line 1001 to 2000 in the SVN list
./get_themes_r2.sh -a -s 1001 -e 2000

# Process themes from line 2001 to 3000 in the SVN list
./get_themes_r2.sh -a -s 2001 -e 3000

# Run with D1 database import enabled
./get_themes_r2.sh -i -w https://your-d1-worker-url.workers.dev
```

### get_themes_r2.sh Example

Example of downloading and mirroring to Cloudflare R2 S3 object store a single specified `generatepress` WordPress theme zip file. The script supports, specificied theme downloads, all themes downloads and also range of lines from themes' SVN list repository.

```bash
time ./get_themes_r2.sh -d                                                                                           

Processing theme: generatepress
[DEBUG] Checking latest version and download link for generatepress
[DEBUG] Latest version for generatepress: 3.5.1
[DEBUG] API download link for generatepress: https://downloads.wordpress.org/theme/generatepress.3.5.1.zip
[DEBUG] Stored version for generatepress: 
[DEBUG] API-provided download link for generatepress: https://downloads.wordpress.org/theme/generatepress.3.5.1.zip
[DEBUG] Constructed download link for generatepress: https://downloads.wordpress.org/theme/generatepress.3.5.1.zip
[DEBUG] Using API-provided download link for generatepress: https://downloads.wordpress.org/theme/generatepress.3.5.1.zip
[DEBUG] Downloading generatepress version 3.5.1 through Cloudflare Worker
[DEBUG] Successfully downloaded generatepress version 3.5.1 from WordPress
[DEBUG] R2 bucket saving for theme zip occurred
Successfully processed generatepress.
Time taken for generatepress: 2.0124 seconds
[DEBUG] Saving theme json metadata for generatepress version 3.5.1
[DEBUG] json metadata for generatepress version 3.5.1 saved (json metadata file already exists)
[DEBUG] Successfully saved json metadata for generatepress.
Theme download process completed.

real    0m3.651s
user    0m0.080s
sys     0m0.036s
```

Re-run with `-c` cache only mode so script only populates Cloudflare R2 S3 object storage bucket without downloading locally which is nearly 85% faster than having to download it locally.

```bash
rm -f /home/nginx/domains/themes.domain.com/public/generatepress.3.5.1.zip 

time ./get_themes_r2.sh -d -c

Processing theme: generatepress
[DEBUG] Latest version for generatepress: 3.5.1
[DEBUG] API download link for generatepress: https://downloads.wordpress.org/theme/generatepress.3.5.1.zip
[DEBUG] Stored version for generatepress: 3.5.1
[DEBUG] API-provided download link for generatepress: https://downloads.wordpress.org/theme/generatepress.3.5.1.zip
[DEBUG] Constructed download link for generatepress: https://downloads.wordpress.org/theme/generatepress.3.5.1.zip
[DEBUG] Using API-provided download link for generatepress: https://downloads.wordpress.org/theme/generatepress.3.5.1.zip
[DEBUG] Downloading generatepress version 3.5.1 through Cloudflare Worker
[DEBUG] Theme generatepress version 3.5.1 cached successfully.
Successfully processed generatepress.
Time taken for generatepress: 0.0999 seconds
[DEBUG] Saving theme json metadata for generatepress version 3.5.1
[DEBUG] Theme metadata for generatepress version 3.5.1 cached successfully.
[DEBUG] Successfully saved json metadata for generatepress.
Theme download process completed.

real    0m0.549s
user    0m0.073s
sys     0m0.027s

ls -lAh /home/nginx/domains/themes.domain.com/public/generatepress.3.5.1.zip 
ls: cannot access '/home/nginx/domains/themes.domain.com/public/generatepress.3.5.1.zip': No such file or directory
```

WordPress themes API 1.0 endpoint doesn't seem to exist, so have to use API 1.2 endpoint to grab the WordPress themes JSON metadata. Here's the official `wordpress.org` themes API 1.2 endpoint result for `generatepress` theme.


```bash
curl -s "https://api.wordpress.org/themes/info/1.2/?action=theme_information&slug=generatepress" | jq -r
{
  "name": "GeneratePress",
  "slug": "generatepress",
  "version": "3.5.1",
  "preview_url": "https://wp-themes.com/generatepress/",
  "author": {
    "user_nicename": "edge22",
    "profile": "https://profiles.wordpress.org/edge22/",
    "avatar": "https://secure.gravatar.com/avatar/dc8bc338c2ebbe1eb9c3cb92ea10486c?s=96&d=monsterid&r=g",
    "display_name": "Tom",
    "author": "Tom Usborne",
    "author_url": "https://tomusborne.com"
  },
  "screenshot_url": "//ts.w.org/wp-content/themes/generatepress/screenshot.png?ver=3.5.1",
  "rating": 100,
  "num_ratings": 1421,
  "reviews_url": "https://wordpress.org/support/theme/generatepress/reviews/",
  "downloaded": 6091807,
  "last_updated": "2024-09-04",
  "last_updated_time": "2024-09-04 14:45:55",
  "creation_time": "2014-05-15 03:45:18",
  "homepage": "https://wordpress.org/themes/generatepress/",
  "sections": {
    "description": "GeneratePress is a lightweight WordPress theme built with a focus on speed and usability. Performance is important to us, which is why a fresh GeneratePress install adds less than 10kb (gzipped) to your page size. We take full advantage of the block editor (Gutenberg), which gives you more control over creating your content. If you use page builders, GeneratePress is the right theme for you. It is completely compatible with all major page builders, including Beaver Builder and Elementor. Thanks to our emphasis on WordPress coding standards, we can boast full compatibility with all well-coded plugins, including WooCommerce. GeneratePress is fully responsive, uses valid HTML/CSS, and is translated into over 25 languages by our amazing community of users. A few of our many features include 60+ color controls, powerful dynamic typography, 5 navigation locations, 5 sidebar layouts, dropdown menus (click or hover), and 9 widget areas. Learn more and check out our powerful premium version at https://generatepress.com"
  },
  "download_link": "https://downloads.wordpress.org/theme/generatepress.3.5.1.zip",
  "tags": {
    "blog": "Blog",
    "buddypress": "BuddyPress",
    "custom-background": "Custom background",
    "custom-colors": "Custom colors",
    "custom-header": "Custom header",
    "custom-menu": "Custom menu",
    "e-commerce": "E-commerce",
    "featured-images": "Featured images",
    "flexible-header": "Flexible header",
    "footer-widgets": "Footer widgets",
    "full-width-template": "Full width template",
    "left-sidebar": "Left sidebar",
    "one-column": "One column",
    "right-sidebar": "Right sidebar",
    "rtl-language-support": "RTL language support",
    "sticky-post": "Sticky post",
    "theme-options": "Theme options",
    "threaded-comments": "Threaded comments",
    "three-columns": "Three columns",
    "translation-ready": "Translation ready",
    "two-columns": "Two columns"
  },
  "requires": "6.1",
  "requires_php": "7.4",
  "is_commercial": true,
  "external_support_url": "https://generatepress.com/support",
  "is_community": false,
  "external_repository_url": ""
}
```

Now here's my local Cloudflare CDN mirrored and R2 S3 object store for `/themes/info/1.0/generatepress.json` JSON metadata where I also modifed and inserted my local mirror copy `download_link_mirror` for the WordPress theme at `https://theme-downloads.mycloudflareproxy_domain.com/generatepress.3.5.1.zip`. For local mirror's themes 1.2 API endpoint, plan to also setup a Cloudflare Worker based bridge like the [WordPress plugins 1.2 API endpoint](#bridge) that uses the saved `/themes/info/1.0/generatepress.json` JSON metadata as a database source - allowing my use case to not require setting up a database.

```bash
curl -s "https://themes-api.mycloudflareproxy_domain.com/themes/info/1.0/generatepress.json" | jq -r '[.download_link, .download_link_mirror]'
[
  "https://downloads.wordpress.org/theme/generatepress.3.5.1.zip",
  "https://theme-downloads.mycloudflareproxy_domain.com/generatepress.3.5.1.zip"
]
```

Full output of local Cloudflare CDN mirrored and R2 S3 object for `themes/info/1.0/generatepress.json`.

```json
curl -s "https://themes-api.mycloudflareproxy_domain.com/themes/info/1.0/generatepress.json" | jq -r
{
  "name": "GeneratePress",
  "slug": "generatepress",
  "version": "3.5.1",
  "preview_url": "https://wp-themes.com/generatepress/",
  "author": {
    "user_nicename": "edge22",
    "profile": "https://profiles.wordpress.org/edge22/",
    "avatar": "https://secure.gravatar.com/avatar/dc8bc338c2ebbe1eb9c3cb92ea10486c?s=96&d=monsterid&r=g",
    "display_name": "Tom",
    "author": "Tom Usborne",
    "author_url": "https://tomusborne.com"
  },
  "screenshot_url": "//ts.w.org/wp-content/themes/generatepress/screenshot.png?ver=3.5.1",
  "rating": 100,
  "num_ratings": 1421,
  "reviews_url": "https://wordpress.org/support/theme/generatepress/reviews/",
  "downloaded": 6094504,
  "last_updated": "2024-09-04",
  "last_updated_time": "2024-09-04 14:45:55",
  "creation_time": "2014-05-15 03:45:18",
  "homepage": "https://wordpress.org/themes/generatepress/",
  "sections": {
    "description": "GeneratePress is a lightweight WordPress theme built with a focus on speed and usability. Performance is important to us, which is why a fresh GeneratePress install adds less than 10kb (gzipped) to your page size. We take full advantage of the block editor (Gutenberg), which gives you more control over creating your content. If you use page builders, GeneratePress is the right theme for you. It is completely compatible with all major page builders, including Beaver Builder and Elementor. Thanks to our emphasis on WordPress coding standards, we can boast full compatibility with all well-coded plugins, including WooCommerce. GeneratePress is fully responsive, uses valid HTML/CSS, and is translated into over 25 languages by our amazing community of users. A few of our many features include 60+ color controls, powerful dynamic typography, 5 navigation locations, 5 sidebar layouts, dropdown menus (click or hover), and 9 widget areas. Learn more and check out our powerful premium version at https://generatepress.com"
  },
  "download_link": "https://downloads.wordpress.org/theme/generatepress.3.5.1.zip",
  "tags": {
    "blog": "Blog",
    "buddypress": "BuddyPress",
    "custom-background": "Custom background",
    "custom-colors": "Custom colors",
    "custom-header": "Custom header",
    "custom-menu": "Custom menu",
    "e-commerce": "E-commerce",
    "featured-images": "Featured images",
    "flexible-header": "Flexible header",
    "footer-widgets": "Footer widgets",
    "full-width-template": "Full width template",
    "left-sidebar": "Left sidebar",
    "one-column": "One column",
    "right-sidebar": "Right sidebar",
    "rtl-language-support": "RTL language support",
    "sticky-post": "Sticky post",
    "theme-options": "Theme options",
    "threaded-comments": "Threaded comments",
    "three-columns": "Three columns",
    "translation-ready": "Translation ready",
    "two-columns": "Two columns"
  },
  "requires": "6.1",
  "requires_php": "7.4",
  "is_commercial": true,
  "external_support_url": "https://generatepress.com/support",
  "is_community": false,
  "external_repository_url": "",
  "download_link_mirror": "https://theme-downloads.mycloudflareproxy_domain.com/generatepress.3.5.1.zip"
}
```

HTTP response headers for Cloudflare CDN cached and R2 S3 object stored JSON metadata files and theme zip file:

```bash
curl -I https://themes-api.mycloudflareproxy_domain.com/themes/info/1.0/generatepress.json
HTTP/2 200 
date: Sun, 13 Oct 2024 11:27:06 GMT
content-length: 2843
etag: "28e63751d8e1bd0dfe28f4c746cf951b"
last-modified: Sun, 13 Oct 2024 10:50:03 GMT
vary: Accept-Encoding
cf-cache-status: HIT
age: 24
expires: Wed, 13 Nov 2024 11:27:06 GMT
cache-control: public, max-age=2678400
accept-ranges: bytes
server: cloudflare
cf-ray: 8d1efa8469e33178-DFW
alt-svc: h3=":443"; ma=86400
```

```bash
curl -I https://theme-downloads.mycloudflareproxy_domain.com/generatepress.3.5.1.zip
HTTP/2 200 
date: Sun, 13 Oct 2024 11:30:10 GMT
content-type: application/zip
content-length: 1065626
etag: "7637f7be37ad9d059fc1599dfe3fc3f5"
last-modified: Sun, 13 Oct 2024 10:50:02 GMT
vary: Accept-Encoding
cf-cache-status: HIT
age: 5
expires: Wed, 13 Nov 2024 11:30:10 GMT
cache-control: public, max-age=2678400
accept-ranges: bytes
server: cloudflare
cf-ray: 8d1efefcafc9477a-DFW
alt-svc: h3=":443"; ma=86400
```

I have a separate script for querying the WordPress Themes API 1.2 endpoint, `query_themes.py` (also have one for `query_plugins.py`) used for quick verification purposes.

```bash
time ./query_themes.py -pages 1 -perpage 10 | jq -r
[
  {
    "name": "Astra",
    "slug": "astra",
    "version": "4.8.3",
    "tested": "5.3",
    "requires_php": "5.3",
    "downloaded": 13518954,
    "rating": 98,
    "last_updated": "2024-10-07",
    "download_link": "https://downloads.wordpress.org/theme/astra.4.8.3.zip"
  },
  {
    "name": "Twenty Twenty",
    "slug": "twentytwenty",
    "version": "2.7",
    "tested": "4.7",
    "requires_php": "5.2.4",
    "downloaded": 10085058,
    "rating": 88,
    "last_updated": "2024-07-16",
    "download_link": "https://downloads.wordpress.org/theme/twentytwenty.2.7.zip"
  },
  {
    "name": "Twenty Twenty-One",
    "slug": "twentytwentyone",
    "version": "2.3",
    "tested": "5.3",
    "requires_php": "5.6",
    "downloaded": 8521207,
    "rating": 84,
    "last_updated": "2024-07-16",
    "download_link": "https://downloads.wordpress.org/theme/twentytwentyone.2.3.zip"
  },
  {
    "name": "Hello Elementor",
    "slug": "hello-elementor",
    "version": "3.1.1",
    "tested": "6.0",
    "requires_php": "7.4",
    "downloaded": 8173562,
    "rating": 88,
    "last_updated": "2024-07-30",
    "download_link": "https://downloads.wordpress.org/theme/hello-elementor.3.1.1.zip"
  },
  {
    "name": "OceanWP",
    "slug": "oceanwp",
    "version": "3.6.1",
    "tested": "5.6",
    "requires_php": "7.4",
    "downloaded": 7781758,
    "rating": 98,
    "last_updated": "2024-10-08",
    "download_link": "https://downloads.wordpress.org/theme/oceanwp.3.6.1.zip"
  },
  {
    "name": "Twenty Twenty-Two",
    "slug": "twentytwentytwo",
    "version": "1.8",
    "tested": "5.9",
    "requires_php": "5.6",
    "downloaded": 5890879,
    "rating": 64,
    "last_updated": "2024-07-16",
    "download_link": "https://downloads.wordpress.org/theme/twentytwentytwo.1.8.zip"
  },
  {
    "name": "Twenty Twenty-Three",
    "slug": "twentytwentythree",
    "version": "1.5",
    "tested": "6.1",
    "requires_php": "5.6",
    "downloaded": 4229009,
    "rating": 68,
    "last_updated": "2024-07-16",
    "download_link": "https://downloads.wordpress.org/theme/twentytwentythree.1.5.zip"
  },
  {
    "name": "Kadence",
    "slug": "kadence",
    "version": "1.2.9",
    "tested": "6.3",
    "requires_php": "7.4",
    "downloaded": 3018261,
    "rating": 98,
    "last_updated": "2024-08-20",
    "download_link": "https://downloads.wordpress.org/theme/kadence.1.2.9.zip"
  },
  {
    "name": "Twenty Twenty-Four",
    "slug": "twentytwentyfour",
    "version": "1.2",
    "tested": "6.4",
    "requires_php": "7.0",
    "downloaded": 1665013,
    "rating": 78,
    "last_updated": "2024-07-16",
    "download_link": "https://downloads.wordpress.org/theme/twentytwentyfour.1.2.zip"
  },
  {
    "name": "CentralNews",
    "slug": "centralnews",
    "version": "1.2.12",
    "tested": false,
    "requires_php": "5.6",
    "downloaded": 13485,
    "rating": 100,
    "last_updated": "2024-10-11",
    "download_link": "https://downloads.wordpress.org/theme/centralnews.1.2.12.zip"
  }
]

real    0m4.856s
user    0m0.241s
sys     0m0.023s
```

### WordPress Themes Cloudflare CDN Benchmarks

Like the [WordPress plugins mirror benchmarks](#wget-download-speed-test), Cloudflare CDN is way faster. Cloudflare CDN is up to 28.6x faster for download speed and 87% faster in terms of latency response times.

| Source | Run | Download Speed (MB/s) |
|--------|-----|------------------------|
| Cloudflare CDN | 1 | 209 |
| Cloudflare CDN | 2 | 220 |
| WordPress.org | 1 | 7.69 |
| WordPress.org | 2 | 7.94 |

Cloudflare CDN mirrored download run 1:

```bash
wget -O /dev/null -S https://theme-downloads.mycloudflareproxy_domain.com/generatepress.3.5.1.zip
--2024-10-13 06:45:57--  https://theme-downloads.mycloudflareproxy_domain.com/generatepress.3.5.1.zip
Resolving theme-downloads.mycloudflareproxy_domain.com (theme-downloads.mycloudflareproxy_domain.com)... 104.xxx.xxx.xxx, 104.xxx.xxx.xxx, 2606:xxx, ...
Connecting to theme-downloads.mycloudflareproxy_domain.com (theme-downloads.mycloudflareproxy_domain.com)|104.xxx.xxx.xxx|:443... connected.
HTTP request sent, awaiting response... 
  HTTP/1.1 200 OK
  Date: Sun, 13 Oct 2024 11:45:58 GMT
  Content-Type: application/zip
  Content-Length: 1065626
  Connection: keep-alive
  ETag: "7637f7be37ad9d059fc1599dfe3fc3f5"
  Last-Modified: Sun, 13 Oct 2024 10:50:02 GMT
  Vary: Accept-Encoding
  CF-Cache-Status: HIT
  Age: 953
  Expires: Wed, 13 Nov 2024 11:45:58 GMT
  Cache-Control: public, max-age=2678400
  Accept-Ranges: bytes
  Server: cloudflare
  CF-RAY: 8d1f1621c9916b57-DFW
  alt-svc: h3=":443"; ma=86400
Length: 1065626 (1.0M) [application/zip]
Saving to: ‘/dev/null’

/dev/null                                100%[===============================================================================>]   1.02M  --.-KB/s    in 0.005s  

2024-10-13 06:45:58 (209 MB/s) - ‘/dev/null’ saved [1065626/1065626]
```

run 2:

```bash
wget -O /dev/null -S https://theme-downloads.mycloudflareproxy_domain.com/generatepress.3.5.1.zip
--2024-10-13 06:50:02--  https://theme-downloads.mycloudflareproxy_domain.com/generatepress.3.5.1.zip
Resolving theme-downloads.mycloudflareproxy_domain.com (theme-downloads.mycloudflareproxy_domain.com)... 104.xxx.xxx.xxx, 104.xxx.xxx.xxx
Connecting to theme-downloads.mycloudflareproxy_domain.com (theme-downloads.mycloudflareproxy_domain.com)|104.xxx.xxx.xxx|:443... connected.
HTTP request sent, awaiting response... 
  HTTP/1.1 200 OK
  Date: Sun, 13 Oct 2024 11:50:03 GMT
  Content-Type: application/zip
  Content-Length: 1065626
  Connection: keep-alive
  ETag: "7637f7be37ad9d059fc1599dfe3fc3f5"
  Last-Modified: Sun, 13 Oct 2024 10:50:02 GMT
  Vary: Accept-Encoding
  CF-Cache-Status: HIT
  Age: 1198
  Expires: Wed, 13 Nov 2024 11:50:03 GMT
  Cache-Control: public, max-age=2678400
  Accept-Ranges: bytes
  Server: cloudflare
  CF-RAY: 8d1f1c1cfd6e4857-DFW
  alt-svc: h3=":443"; ma=86400
Length: 1065626 (1.0M) [application/zip]
Saving to: ‘/dev/null’

/dev/null                                100%[===============================================================================>]   1.02M  --.-KB/s    in 0.005s  

2024-10-13 06:50:03 (220 MB/s) - ‘/dev/null’ saved [1065626/1065626]
```

`wordpress.org` download run 1:

```bash
wget -O /dev/null -S https://downloads.wordpress.org/theme/generatepress.3.5.1.zip
--2024-10-13 06:44:33--  https://downloads.wordpress.org/theme/generatepress.3.5.1.zip
Resolving downloads.wordpress.org (downloads.wordpress.org)... 198.143.164.250
Connecting to downloads.wordpress.org (downloads.wordpress.org)|198.143.164.250|:443... connected.
HTTP request sent, awaiting response... 
  HTTP/1.1 200 OK
  Server: nginx
  Date: Sun, 13 Oct 2024 11:44:34 GMT
  Content-Type: application/zip
  Content-Length: 1065626
  Connection: close
  Content-Disposition: attachment; filename=generatepress.3.5.1.zip
  Content-MD5: 7637f7be37ad9d059fc1599dfe3fc3f5
  Access-Control-Allow-Methods: GET, HEAD
  Access-Control-Allow-Origin: *
  Alt-Svc: h3=":443"; ma=86400
  X-nc: MISS ord 4
Length: 1065626 (1.0M) [application/zip]
Saving to: ‘/dev/null’

/dev/null                                100%[===============================================================================>]   1.02M  --.-KB/s    in 0.1s    

2024-10-13 06:44:34 (7.69 MB/s) - ‘/dev/null’ saved [1065626/1065626]
```

`wordpress.org` download run 2:

```bash
wget -O /dev/null -S https://downloads.wordpress.org/theme/generatepress.3.5.1.zip
--2024-10-13 06:48:12--  https://downloads.wordpress.org/theme/generatepress.3.5.1.zip
Resolving downloads.wordpress.org (downloads.wordpress.org)... 198.143.164.250
Connecting to downloads.wordpress.org (downloads.wordpress.org)|198.143.164.250|:443... connected.
HTTP request sent, awaiting response... 
  HTTP/1.1 200 OK
  Server: nginx
  Date: Sun, 13 Oct 2024 11:48:13 GMT
  Content-Type: application/zip
  Content-Length: 1065626
  Connection: close
  Content-Disposition: attachment; filename=generatepress.3.5.1.zip
  Content-MD5: 7637f7be37ad9d059fc1599dfe3fc3f5
  Access-Control-Allow-Methods: GET, HEAD
  Access-Control-Allow-Origin: *
  Alt-Svc: h3=":443"; ma=86400
  X-nc: MISS ord 4
Length: 1065626 (1.0M) [application/zip]
Saving to: ‘/dev/null’

/dev/null                                100%[===============================================================================>]   1.02M  --.-KB/s    in 0.1s    

2024-10-13 06:48:13 (7.94 MB/s) - ‘/dev/null’ saved [1065626/1065626]
```

<a name="curltimesthemes"></a>
Comparing [`curltimes.sh`](https://github.com/centminmod/curltimes) for both

| Metric | Run | Cloudflare CDN (s) | Original WordPress (s) | Difference (s) | Percentage Difference (%) |
|--------|-----|---------------------|------------------------|----------------|---------------------------|
| DNS Lookup | 1 | 0.017868 | 0.012266 | -0.005602 | -45.67% |
|            | 2 | 0.016065 | 0.012368 | -0.003697 | -29.89% |
|            | 3 | 0.012104 | 0.011892 | -0.000212 | -1.78% |
| **DNS Avg** |  | **0.015346** | **0.012175** | **-0.003171** | **-26.05%** |
| Connect | 1 | 0.019249 | 0.066082 | 0.046833 | 70.87% |
|         | 2 | 0.017640 | 0.066095 | 0.048455 | 73.31% |
|         | 3 | 0.013583 | 0.065603 | 0.052020 | 79.29% |
| **Connect Avg** |  | **0.016824** | **0.065927** | **0.049103** | **74.48%** |
| SSL | 1 | 0.040410 | 0.190844 | 0.150434 | 78.83% |
|     | 2 | 0.039722 | 0.188994 | 0.149272 | 78.98% |
|     | 3 | 0.036624 | 0.188560 | 0.151936 | 80.58% |
| **SSL Avg** |  | **0.038919** | **0.189466** | **0.150547** | **79.46%** |
| TTFB | 1 | 0.074087 | 0.247062 | 0.172975 | 70.01% |
|      | 2 | 0.082480 | 0.245153 | 0.162673 | 66.36% |
|      | 3 | 0.079275 | 0.502297 | 0.423022 | 84.22% |
| **TTFB Avg** |  | **0.078614** | **0.331504** | **0.252890** | **76.29%** |
| Total Time | 1 | 0.087251 | 0.631285 | 0.544034 | 86.18% |
|            | 2 | 0.095769 | 0.655155 | 0.559386 | 85.38% |
|            | 3 | 0.092491 | 0.855737 | 0.763246 | 89.19% |
| **Total Avg** |  | **0.091837** | **0.714059** | **0.622222** | **87.14%** |

Cloudflare CDN cached

```bash
./curltimes.sh json https://theme-downloads.mycloudflareproxy_domain.com/generatepress.3.5.1.zip
curl 7.76.1
TLSv1.2 ECDHE-ECDSA-CHACHA20-POLY1305
Connected to theme-downloads.mycloudflareproxy_domain.com (104.xxx.xxx.xxx) port 443 (#0)
Cloudflare proxied https://theme-downloads.mycloudflareproxy_domain.com/generatepress.3.5.1.zip
Sample Size: 3

DNS,Connect,SSL,Wait,TTFB,Total Time
{
        "time_dns":             0.017868,
        "time_connect":         0.019249,
        "time_appconnect":      0.040410,
        "time_pretransfer":     0.040497,
        "time_ttfb":            0.074087,
        "time_total":           0.087251
}{
        "time_dns":             0.016065,
        "time_connect":         0.017640,
        "time_appconnect":      0.039722,
        "time_pretransfer":     0.039789,
        "time_ttfb":            0.082480,
        "time_total":           0.095769
}{
        "time_dns":             0.012104,
        "time_connect":         0.013583,
        "time_appconnect":      0.036624,
        "time_pretransfer":     0.036688,
        "time_ttfb":            0.079275,
        "time_total":           0.092491
}
```

Original Wordpress plugin

```bash
./curltimes.sh json https://downloads.wordpress.org/theme/generatepress.3.5.1.zip
TLSv1.2 ECDHE-ECDSA-CHACHA20-POLY1305
Connected to downloads.wordpress.org (198.143.164.250) port 443 (#0)
Sample Size: 3

DNS,Connect,SSL,Wait,TTFB,Total Time
{
        "time_dns":             0.012266,
        "time_connect":         0.066082,
        "time_appconnect":      0.190844,
        "time_pretransfer":     0.190920,
        "time_ttfb":            0.247062,
        "time_total":           0.631285
}{
        "time_dns":             0.012368,
        "time_connect":         0.066095,
        "time_appconnect":      0.188994,
        "time_pretransfer":     0.189173,
        "time_ttfb":            0.245153,
        "time_total":           0.655155
}{
        "time_dns":             0.011892,
        "time_connect":         0.065603,
        "time_appconnect":      0.188560,
        "time_pretransfer":     0.188627,
        "time_ttfb":            0.502297,
        "time_total":           0.855737
}
```

### WordPress Themes Mirrors

Running the full WordPress themes mirroring resulted in a total of 13,021 WordPress themes occupying a total 17.16GB disk size in Cloudflare R2 S3 object storage bucket. There are currently 27,494 WordPress themes listed in Wordpress themes SVN repository at https://themes.svn.wordpress.org/ so 14,448 WordPress themes in SVN repo that are not open/published on `wordpress.org`. 

Local WordPress themes download size and counts:

```bash
du -sh /home/nginx/domains/themes.domain.com/public/
18G     /home/nginx/domains/themes.domain.com/public/

du -s /home/nginx/domains/themes.domain.com/public/
18022760        /home/nginx/domains/themes.domain.com/public/

ls -lAh /home/nginx/domains/themes.domain.com/public | wc -l
13021
```

#### WordPress Themes Mirror Screenshots

WordPress Themes in Cloudflare R2 S3 Bucket viewed via S3Browser Windows app:

![WordPress Themes in Cloudflare R2 S3 Bucket](/screenshots/allplugins-get_themes_r2_s3browser-listing-themes-01.png)

Here's the Cloudflare R2 S3 bucket's metrics after the full initial Wordpress themes mirroring and saving to Cloudflare R2 S3 bucket. This full mirror initial session was ran in single threaded mode to see how long it takes. As scripts support parallel threads mode, will see how fast it is in future. I did wait a few days after the mirror run to grab these metrics so had to expand the metric duration to the past week.

#### WordPress Themes Mirror Cloudflare R2 S3 Metrics Screenshots

For WordPress themes mirror R2 S3 bucket reporting at 18.43GB in size with 12.99K Class A writes and 13.24K Class B reads.

![WordPress Themes in Cloudflare R2 S3 Bucket metrics](/screenshots/cf-r2-themes-dashboard-01.png)

![WordPress Themes in Cloudflare R2 S3 Bucket metrics](/screenshots/cf-r2-themes-dashboard-02.png)

![WordPress Themes in Cloudflare R2 S3 Bucket metrics](/screenshots/cf-r2-themes-dashboard-03.png)

## WordPress Secret Keys API Generator

Not WordPress plugin related, but also setup a separate Cloudflare Worker that replicates (not proxy) the API for secret keys generator for wp-config.php.

Original

* https://api.wordpress.org/secret-key/1.0/
* https://api.wordpress.org/secret-key/1.1/
* https://api.wordpress.org/secret-key/1.1/salt/

Cloudflare Worker

* https://api.mycloudflareproxy_domain.com/secret-key/1.0/
* https://api.mycloudflareproxy_domain.com/secret-key/1.1/
* https://api.mycloudflareproxy_domain.com/secret-key/1.1/salt/

```bash
curl -s https://api.mycloudflareproxy_domain.com/secret-key/1.0/

define('SECRET_KEY', '(r&lIZV-.8/@jc?t[>D~xZTPuIq.B2GFF>Mc.-Gu%+yg36YMjyqJe.:4WYo4<~ig');
```

```bash
curl -s https://api.mycloudflareproxy_domain.com/secret-key/1.1/

define('AUTH_KEY', 'x+%Hff>o}T^jaNbyh*[S`C&wTOF Ygqk<iY:5hgEXQ$M+/Qe$Nct$A6a7qm>u]R$');
define('SECURE_AUTH_KEY', '4qpO5+&mC#0fT?sAb2p[D+qw`f<%{w56s4*(8pY`Rd{}<~N*wS6EUGj-?/q@M,B1');
define('LOGGED_IN_KEY', 'qMj7-!R2B,L>wQN9_/){8l(:1Ur@&m1]/&C=4 G<!|DUC}>ZgKj4H(#/ha[`#7w~');
define('NONCE_KEY', '7(eOLM[3>8E2)J,torRGm?mhB@EW136oK+UKqh3B-#TiJWE{TA_*lf0+zFcz@X5^');
```

```bash
curl -s https://api.mycloudflareproxy_domain.com/secret-key/1.1/salt/

define('AUTH_KEY', 'J0:Lo?H4}F&s(Qj}seebJ`H,R!ntuELcmn2aIx(&sc5GabjGU(h-dp^IQ6MfnKqI');
define('SECURE_AUTH_KEY', 'ywn%1U3;NySz3jT+m|Y;Xin{+Za7Uh^8][VIVwW,?t!{:nksrC=3|.UFQG^Zs&uA');
define('LOGGED_IN_KEY', 'm<[y_4sW$Kuc>E$3^/q >2Dg{SV@9L*~a&lW#<_.Rr$5cnz:6j9i;@NYy?C[%WF-');
define('NONCE_KEY', 'jn$?ao_n+gk+2I?s}Pv13bd}:yr$-1$XtO??4kBvsl5:a6ZylXOn3=NdGx%+*lOp');
define('AUTH_SALT', 'MA<4uGqc`biF-%d.|@!lKEeB<`V)q0rZn->ZvEAf#s9Ysqg=Wt%>0<@Ar-*TnPS?');
define('SECURE_AUTH_SALT', 'yW})O/%Q$YAW|1~[Y2?yinMw &U<<VW*_{0,}leIwA~?vaIP_K-~s[iB;=&@YI/7');
define('LOGGED_IN_SALT', 'gt,yqP|J7pA8yd)~JhSA>[L&p1r$G/H=(9P(CWqR=a([N&h<K~p`5QhzD59rG5uq');
define('NONCE_SALT', 'X_c%R1Aqaw:.mO^Mr(J}[?ly-dScByC*0v*gafH>C#ptX~{Y?jU?C&C:%)GV;J}~');
```