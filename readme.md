# WordPress Plugin Mirror Downloader: User Documentation

## Table of Contents
1. [Introduction](#introduction)
   * [SVN Mirroring](#svn-mirroring)
     * [Using csync2 cluster](#using-csync2-cluster)
   * [Key Components](#key-components)
   * [System Advantages](#system-advantages)
   * [Cloudflare Related Costs](#cloudflare-related-costs)
     * [Cloudflare R2 GraphQL Metrics](#cloudflare-r2-graphql-metrics)
2. [System Overview](#system-overview)
3. [Examples](#examples)
   * [Mirrored WordPress Plugin API End Point](#mirrored-wordpress-plugin-api-end-point)
   * [Cached Plugin](#cached-plugin)
   * [wget download speed test](#wget-download-speed-test)
4. [Screenshots](#screenshots)

## Introduction

The WordPress Plugin Mirror Downloader is a sophisticated system designed to efficiently download, cache, and manage WordPress plugins. It leverages Cloudflare's edge computing capabilities and object storage to create a high-performance, scalable solution for plugin management. If you are a open source project, Cloudflare has expanded it's free support offerings via Cloudflare Project Alexandria https://blog.cloudflare.com/expanding-our-support-for-oss-projects-with-project-alexandria/. 

Given [Cloudflare CDN cached WordPress plugin download benchmarks speeds](#cached-plugin) with 43x times faster downlaod speed and 82% faster latency than plugin served from Wordpress.org, Matt Mullenweg should take up Cloudflare CEO, Matthew Prince's [offer of donating capacity to Wordpress.org](https://x.com/eastdakota/status/1841154152006627663?t=L0e-TL1cPhkgckxPDG6nvg&s=19). Would dramatically cut Wordpres.org's infrastructure running costs and speed up file downloads. :sunglasses:

The below POC method only focuses on mirroring and downloading WordPress Plugin zip files itself instead of mirroring the entire WordPress plugin SVN repository. The reason is SVN repositories contain the history of all commits and versions for a WordPress plugin so involve alot more files and data size/transfer is larger. 

Disclaimer, I am a Cloudflare customer since 2011 and official Cloudflare community MVP since 2018 (non-paid similar to how Microsoft MVP program operates) using Cloudflare Free, Pro, Business and Enterprise plans.

### SVN Mirroring

Prior to this POC, I did try POC for SVN mirroring at https://gist.github.com/centminmod/003654673b3c6b11e10edc9353551fd2 and for test 53 WordPress plugins, total disk space to mirror them was approximately 40GB in size. So you will need alot less disk resources and bandwidth if you only focus on WordPress plugin zip files and not the entire SVN repository. In comparison with below mirroring of zip files only, the size for test run of 563 WordPress plugin zip files download and cache into Cloudflare R2 S3 object storage was ~1.27GB in size for zip files and ~18MB for plugin JSON metadata files. Rough maths for 563 plugins taking ~1.3GB storage space. So for 103K plugins would be ~238GB total storage space which will be beyond Cloudflare R2 S3 object storage's Forever Free tier of 10GB/month storage. So there would be additional storage costs - unless you are an open source project under Cloudflare Project Alexandria.

You can also leverage Cloudflare R2 as mounted Linux FUSE mount via JuiceFS which caches file metadata for better performance and allows you to mount Cloudflare R2 S3 mounts in sharded mounts as well. See my write up and benchmarks for Cloudflare R2 + JuiceFS https://github.com/centminmod/centminmod-juicefs.

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

2. **Cloudflare R2 Storage**: An S3-compatible object storage system used to cache plugin ZIP files and metadata JSON. Cloudflare R2 S3 object storage has free egress bandwidth costs so you only pay for object storage and read/writes to object storage.

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

- **WordPress Plugin 1.2 API Bridge Worker**: An additional WordPress Plugin API Bridge Worker is created using a separate Cloudflare Worker. It is designed to bridge the gap between the WordPress Plugin API 1.0 and 1.2 versions `https://api.wordpress.org/plugins/info/1.0` vs `https://api.wordpress.org/plugins/info/1.2`. It allows clients to query plugin information using the 1.2 API format while fetching data from either a mirrored 1.0 API endpoint or the official WordPress.org 1.0 API, providing flexibility and reliability in data retrieval

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

### Cloudflare Related Costs

Cloudflare CDN is free of charge, so only costs will be related to using Cloudflare Workers and Cloudflare R2 S3 object storage as outlined below.

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

The below first three examples have much higher R2 write estimations. With forth example probably closer to WordPress Plugin zip file mirroring.

#### Example 1 - cost calculation for:

* 250GB of R2 storage with 5 million write and 25 million read operations 
* Cloudflare Worker for 10 million requests averaging 1.8ms CPU time

1. R2 Storage Costs:
   - Storage: 250 GB at $0.015 per GB-month
   - 250 * $0.015 = $3.75 per month

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

- R2 Storage: $3.75
- R2 Write Operations: $22.50
- R2 Read Operations: $9.00
- Cloudflare Worker usage: $0 (within included limits)
- Cloudflare Worker Subscription fee: $5

Total monthly cost: $40.25 per month

#### Example 2 - large cost calculation for:

- 2500GB of R2 storage
- 50 million write operations and 250 million read operations on R2
- Cloudflare Worker handling 100 million requests, averaging 2.5ms CPU time per request

1. R2 Storage Costs
   - Storage: 2500 GB at $0.015 per GB-month
   - 2500 * $0.015 = $37.50 per month

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

- R2 Storage: $37.50
- R2 Write Operations: $225.00
- R2 Read Operations: $90.00
- Cloudflare Worker Requests: $27.00
- Cloudflare Worker CPU time: $4.40
- Cloudflare Worker Subscription fee: $5.00

Total Monthly Cost: $388.90 per month

#### Example 3 - extra large cost calculation for:

- 2000GB of R2 storage
- 500 million write operations and 250 million read operations on R2
- Cloudflare Worker handling 1 billion requests, averaging 3ms CPU time per request

1. R2 Storage Costs
   - Storage: 5000 GB at $0.015 per GB-month
   - 5000 * $0.015 = $75.00 per month

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

- R2 Storage: $75.00
- R2 Write Operations: $2,250.00
- R2 Read Operations: $900.00
- Cloudflare Worker Requests: $297.00
- Cloudflare Worker CPU time: $59.40
- Cloudflare Worker Subscription fee: $5.00

Total Monthly Cost: $3,586.40 per month

#### Example 4 - more realistic R2 Writes:

There are ~103K Wordpress plugins as of writing. If each plugin would consistently release a new version per day for 30 days, there would be 2x 103,000 R2 writes/day - one for R2 write for zip file and one for R2 write for JSON metadata file = 206,000/day = 6.18 million R2 writes per month. Obviously, not every plugin would be releasing a new version every day for an entire month.

- 250GB of R2 storage with 6.18 million write and 10 billion read operations. Note, if you implement Cloudflare CDN cache using [Cache Rules](https://developers.cloudflare.com/cache/how-to/cache-rules/) in front of R2 stored files, you won't get anywhere near 10 bliion read operations in reality. Example of Cloudflare CDN cached mirrored WordPress plugin [here](#cached-plugin).
- Cloudflare Worker handling 10 billion requests, averaging 3ms CPU time per request
- R2 read and write operations also haven't accounted for my script's inspection of the R2 buckets' respective contents.

1. R2 Storage Costs
   - Storage: 250 GB at $0.015 per GB-month
   - 250 * $0.015 = $3.75 per month

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

- R2 Storage: $3.75
- R2 Write Operations: $27.81
- R2 Read Operations: $3,600.00 (reduced pricing if implement Cloudflare CDN cache using [Cache Rules](https://developers.cloudflare.com/cache/how-to/cache-rules/) in front of R2 stored files)
- Cloudflare Worker Requests: $2,997.00
- Cloudflare Worker CPU time: $599.40
- Cloudflare Worker Subscription fee: $5.00

Total Monthly Cost: $7,232.96 per month

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
No metrics found.

Checking last 60 minutes...
No metrics found.

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
No metrics found

Last 60 minutes:
No metrics found

Last 120 minutes:
Querying data from 2024-10-03T07:52:46Z to 2024-10-03T09:52:46Z
GetObject: 1
PutObject: 1

Last 360 minutes:
Querying data from 2024-10-03T03:52:46Z to 2024-10-03T09:52:46Z
GetObject: 3
PutObject: 1

Last 720 minutes:
Querying data from 2024-10-02T21:52:47Z to 2024-10-03T09:52:47Z
GetObject: 3
PutObject: 1

Last 1440 minutes:
Querying data from 2024-10-02T09:52:47Z to 2024-10-03T09:52:47Z
GetObject: 1174
PutObject: 312

Last 2880 minutes:
Querying data from 2024-10-01T09:52:47Z to 2024-10-03T09:52:47Z
GetObject: 2245
PutObject: 570

Last 4320 minutes:
Querying data from 2024-09-30T09:52:48Z to 2024-10-03T09:52:48Z
GetObject: 2263
PutObject: 600

Conclusion:
Metrics were found in at least one time range.
Most recent activity: Last 120 minutes

Activity summary:
  PutObject: 600
  GetObject: 2263
  Querying data from 2024-09-30T09:52:48Z to 2024-10-03T09:52:48Z
- Last 4320 minutes:
  PutObject: 570
  GetObject: 2245
  Querying data from 2024-10-01T09:52:47Z to 2024-10-03T09:52:47Z
- Last 2880 minutes:
  PutObject: 312
  GetObject: 1174
  Querying data from 2024-10-02T09:52:47Z to 2024-10-03T09:52:47Z
- Last 1440 minutes:
  PutObject: 1
  GetObject: 3
  Querying data from 2024-10-02T21:52:47Z to 2024-10-03T09:52:47Z
- Last 720 minutes:
  PutObject: 1
  GetObject: 3
  Querying data from 2024-10-03T03:52:46Z to 2024-10-03T09:52:46Z
- Last 360 minutes:
  PutObject: 1
  GetObject: 1
  Querying data from 2024-10-03T07:52:46Z to 2024-10-03T09:52:46Z
- Last 120 minutes:

Trend analysis:
No activity in the last hour. Consider investigating if this is unexpected.
Significant activity in the last 24 hours. Review the 24-hour metrics for a comprehensive overview.
Activity detected in the last 72 hours. Compare with 24-hour and 48-hour metrics to identify trends.
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

### Mirrored WordPress Plugin API End Point

Mirrored and locally cached in Cloudflare R2 bucket plugin JSON metadata where `api.mycloudflareproxy_domain.com` is Cloudflare orange cloud proxy enabled domain zone hostname enabled for Cloudflare R2 bucket public access for `WP_PLUGIN_INFO` bucket referenced in Cloudflare Worker that `get_plugins_r2.sh` talks to.

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

### Cached Plugin

Example of Cloudflare CDN cached plugin compared to Wordpress plugin download.

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

Comparing [`curltimes.sh`](https://github.com/centminmod/curltimes) for both

| Metric | Run | Cloudflare CDN (s) | Original WordPress (s) | Difference (s) |
|--------|-----|---------------------|------------------------|----------------|
| DNS Lookup | 1 | 0.017216 | 0.012032 | -0.005184 |
|            | 2 | 0.017077 | 0.012507 | -0.004570 |
|            | 3 | 0.017610 | 0.012564 | -0.005046 |
| **DNS Avg** |  | **0.017301** | **0.012368** | **-0.004933** |
| Connect | 1 | 0.018738 | 0.065434 | 0.046696 |
|         | 2 | 0.019041 | 0.066386 | 0.047345 |
|         | 3 | 0.019858 | 0.066524 | 0.046666 |
| **Connect Avg** |  | **0.019212** | **0.066115** | **0.046903** |
| SSL | 1 | 0.043311 | 0.187870 | 0.144559 |
|     | 2 | 0.042507 | 0.189180 | 0.146673 |
|     | 3 | 0.042925 | 0.189554 | 0.146629 |
| **SSL Avg** |  | **0.042914** | **0.188868** | **0.145954** |
| TTFB | 1 | 0.100405 | 0.416238 | 0.315833 |
|      | 2 | 0.096419 | 0.248653 | 0.152234 |
|      | 3 | 0.099871 | 0.364501 | 0.264630 |
| **TTFB Avg** |  | **0.098898** | **0.343131** | **0.244233** |
| Total Time | 1 | 0.107244 | 0.665148 | 0.557904 |
|            | 2 | 0.103766 | 0.528787 | 0.425021 |
|            | 3 | 0.108014 | 0.630060 | 0.522046 |
| **Total Avg** |  | **0.106341** | **0.607998** | **0.501657** |

#### Notes:
- Times are in seconds (s)
- Lower times indicate faster performance
- Cloudflare CDN is consistently faster in most metrics, except for initial DNS lookup
- Average total time improvement: 0.501657 seconds (approximately 82.5% faster)

#### Interpretation:
1. **DNS Lookup**: Cloudflare is slightly slower (by ~0.005s), likely due to the additional Cloudflare DNS resolution.
2. **Connect**: Cloudflare is significantly faster (by ~0.047s), possibly due to closer server proximity.
3. **SSL**: Cloudflare performs much better (by ~0.146s), likely due to optimized SSL handshake.
4. **Time to First Byte (TTFB)**: Cloudflare is substantially faster (by ~0.244s), indicating quicker server response.
5. **Total Time**: Cloudflare CDN delivers the file much faster (by ~0.502s), which is a significant improvement.

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