---
group: cloud-guide
title: Set up multiple websites or stores
functional_areas:
  - Cloud
  - Configuration
  - Setup
  - Stores
---

You can configure {{site.data.var.ee}} to have multiple websites or stores, such as an English store, a French store, and a German store. See [Understanding websites, stores, and store views]({{ site.baseurl }}/cloud/configure/configure-best-practices.html#sites).

{:.bs-callout-warning}
Catalog data expands as you increase the number of websites and stores. Depending on your project architecture, the additional stores can lead to a longer indexing process and slower response times for non-cached catalog pages. Adobe recommends that you monitor site performance closely.

The process to set up multiple stores depends on whether you choose to use unique or shared domains.

Multiple stores with unique domains:

```terminal
https://first.store.com/
https://second.store.com/
```
{:.no-copy}

Multiple stores with the same domain:

```terminal
https://store.com/first/
https://store.com/second/
```
{:.no-copy}

{:.bs-callout-tip}
To add a store view to the site base URL, you do not have to create multiple directories. See [Add the store code to the base URL][addstorecode] in the _Config Guide_.

## Add New Domains

Custom domains can be added to Pro Staging and any Production environment; they cannot be added to Integration environments.

The process to add a domain depends on the type of Cloud account:

-  For Pro Staging and Production

   Add the new domain to Fastly, see [Manage domains][], or open a support ticket to request assistance. In addition, you must open a Support ticket to request new domains to be added to a cluster.

-  For Starter Production only

   Add the new domain to Fastly, see [Manage domains][], or open a support ticket to request assistance. In addition, you must add the new domain to the **Domains** tab in the Project Web Interface: `https://<zone>.magento.cloud/projects/<project-ID>/edit`

## Configure local installation

To configure your local installation to use multiple stores, see [Multiple websites or stores][config-multiweb] in the _Config Guide_.

After successfully creating and testing the local installation to use multiple stores, you must prepare your Integration environment:

1. **Configure routes or locations**—specify how incoming URLs are handled by {{site.data.var.ee}}
   -  [Routes for separate domains](#routes)
   -  [Locations for shared domains](#locations)
1. **Set up websites, stores, and store views**—configure using the {{site.data.var.ee}} Admin UI
1. **Modify variables**—specify the values of the `MAGE_RUN_TYPE` and `MAGE_RUN_CODE` variables in the `magento-vars.php` file
1. **Deploy and test environments**—deploy and test the `integration` branch

{:.bs-callout-tip}
You can also use a local {{site.data.var.mcd-prod}} environment to set up multiple websites or stores. See the Cloud Docker instructions to [Set up multiple websites or stores]({{site.baseurl}}/cloud/docker/docker-multi-website.html).

### Configure routes for separate domains {#routes}

Routes define how to process incoming URLs. Multiple stores with unique domains require you to define each domain in the `routes.yaml` file. The way you configure routes depends on how you want your site to operate.

{:.bs-callout-info}
For Pro, if your route configurations are not setting properly, you can create a [Support ticket]({{ site.baseurl }}/cloud/trouble/trouble.html) and include "enable self-service routes" in your request.

{:.procedure}
To configure routes in an integration environment:

1. On your local workstation, open the `.magento/routes.yaml` file in a text editor.

1. Define the domain and subdomains. The `mymagento` upstream value is the same value as the name property in the `.magento.app.yaml` file.

   ```yaml
   "http://{default}/":
       type: upstream
       upstream: "mymagento:http"

   "http://<second-site>.{default}/":
       type: upstream
       upstream: "mymagento:http"
   ```

1. Save your changes to the `routes.yaml` file.

1. Continue to the [_Set up websites, stores, and store views_ section](#set-stores).

### Configure locations for shared domains {#locations}

Where the routes configuration defines how the URLs are processed, the `web` property in the `.magento.app.yaml` file defines how your application is exposed to the web. Web _locations_ allow more granularity for incoming requests. For example, if your domain is `store.com`, you can use `/first` (default site) and `/second` for requests to two different stores that share a domain.

{:.procedure}
To configure a new web location:

1. Create an alias for the root (`/`). In this example, the alias is `&app` on line 3.

   ```yaml
   web:
       locations:
           "/": &app
               # The public directory of the app, relative to its root.
               root: "pub"
               passthru: "/index.php"
               index:
               - index.php
               ...
   ```

1. Create a pass for the website (`/website`) and reference the root using the alias from the previous step.

   The alias allows `website` to access values from the root location. In this example, the website pass is on line 21.

   ```yaml
   web:
       locations:
           "/": &app
               # The public directory of the app, relative to its root.
               root: "pub"
               passthru: "/index.php"
               index:
               - index.php
               ...
           "/media":
               root: "pub/media"
               ...
           "/static":
               root: "pub/static"
               allow: true
               scripts: false
               passthru: "/front-static.php"
               rules:
                   ^/static/version\d+/(?<resource>.*)$:
                       passthru: "/static/$resource"
           "/<website>": *app
             ...
   ```

1. Continue to the [_Set up websites, stores, and store views_ section](#set-stores).

{:.procedure}
To configure a location with a different directory:

1. Create an alias for the root (`/`) and for the static (`/static`) locations.

   ```yaml
   web:
       locations:
           "/": &root
               # The public directory of the app, relative to its root.
               root: "pub"
               passthru: "/index.php"
               index:
               - index.php
               ...
           "/static": &static
               root: "pub/static"
   ```

1. Create a pass through for the `index.php` file.

   ```yaml
   web:
       locations:
           "/": &root
               # The public directory of the app, relative to its root.
               root: "pub"
               passthru: "/index.php"
               index:
               - index.php
               ...
           "/media":
               root: "pub/media"
               ...
           "/static": &static
               root: "pub/static"
               allow: true
               scripts: false
               passthru: "/front-static.php"
               rules:
                   ^/static/version\d+/(?<resource>.*)$:
                       passthru: "/static/$resource"
           "/<website>":
               <<: *root
               passthru: "website/index.php"
           "/<website>/static": *static
             ...
   ```

### Set up websites, stores, and store views {#set-stores}

In the _Admin UI_, set up your {{site.data.var.ee}} **Websites**, **Stores**, and **Store Views**. See [Set up multiple websites, stores, and store views in the Admin]({{ site.baseurl }}/guides/v2.3/config-guide/multi-site/ms_websites.html).

It is important to use the same name and Code of your websites, stores, and store views from your Admin when you set up your local installation. You need these values when you update the `magento-vars.php` file.

### Modify variables

Instead of configuring an NGINX virtual host, pass the `MAGE_RUN_CODE` and `MAGE_RUN_TYPE` variables using the `magento-vars.php` file in your project root directory.

{:.procedure}
To pass variables using the `magento-vars.php` file:

1. Open the `magento-vars.php` file in a text editor.

   The [default `magento-vars.php` file](https://github.com/magento/magento-cloud/blob/master/magento-vars.php) should look like the following:

   ```php
   <?php
   // enable, adjust and copy this code for each store you run
   // Store #0, default one
   //if (isHttpHost("example.com")) {
   //    $_SERVER["MAGE_RUN_CODE"] = "default";
   //    $_SERVER["MAGE_RUN_TYPE"] = "store";
   //}
   function isHttpHost($host)
   {
       if (!isset($_SERVER['HTTP_HOST'])) {
           return false;
       }
       return $_SERVER['HTTP_HOST'] === $host;
   }
   ```
   {:.no-copy}

1. Move the commented `if` block so that it is _after_ the `function` block and no longer commented.

   ```php
   <?php
   // enable, adjust and copy this code for each store you run
   // Store #0, default one

   function isHttpHost($host)
   {
       if (!isset($_SERVER['HTTP_HOST'])) {
           return false;
       }
       return $_SERVER['HTTP_HOST'] ===  $host;
   }

   if (isHttpHost("example.com")) {
       $_SERVER["MAGE_RUN_CODE"] = "default";
       $_SERVER["MAGE_RUN_TYPE"] = "store";
   }
   ```

1. Replace the following values in the `if (isHttpHost("example.com"))` block:
   -  `example.com`—with the base URL of your _website_
   -  `default`—with the unique CODE for your _website_ or _store view_
   -  `store`—with one of the following values:
      -  `website`—load the _website_ in the storefront
      -  `store`—load a _store view_ in the storefront

   For multiple sites using unique domains:

   ```php
   <?php
   function isHttpHost($host)
   {
       if (!isset($_SERVER['HTTP_HOST'])) {
       return false;
       }
       return $_SERVER['HTTP_HOST'] ===  $host;
   }

   if (isHttpHost("second.store.com")) {
       $_SERVER["MAGE_RUN_CODE"] = "<second-site>";
       $_SERVER["MAGE_RUN_TYPE"] = "website";
   }elseif (isHttpHost("store.com")){
       $_SERVER["MAGE_RUN_CODE"] = "base";
       $_SERVER["MAGE_RUN_TYPE"] = "website";
   }
   ```

   For multiple sites with the same domain, you have to check the _host_ and the _URI_:

   ```php
   <?php
   function isHttpHost($host)
   {
       if (!isset($_SERVER['HTTP_HOST'])) {
       return false;
       }
       return $_SERVER['HTTP_HOST'] ===  $host;
   }

   if (isHttpHost("store.com")) {
      $code = "base";
      $type = "website";

      $uri = explode('/', $_SERVER['REQUEST_URI']);
      if (isset($uri[1]) && $uri[1] == 'second') {
        $code = '<second-site>';
        $type = 'website';
      }
      $_SERVER["MAGE_RUN_CODE"] = $code;
      $_SERVER["MAGE_RUN_TYPE"] = $type;
   }
   ```

1. Save your changes to the `magento-vars.php` file.

### Deploy and test on the Integration server

Push your changes to your {{site.data.var.ece}} Integration environment and test your site.

1. Add, commit, and push code changes to the remote branch.

   ```bash
   git add -A && git commit -m "Implement multiple sites" && git push origin <branch-name>
   ```

1. Wait for deployment to complete.

1. After deployment, open your Store URL in a web browser.

   With a unique domain, use the format: `http://<magento-run-code>.<site-URL>`

   For example, `http://french.master-name-projectID.us.magentosite.cloud/`

   With a shared domain, use the format: `http://<site-URL>/<magento-run-code>`

   For example, `http://master-name-projectID.us.magentosite.cloud/french/`

1. Test your site thoroughly and merge the code to the `integration` branch for further deployment.

## Deploy to Staging and Production

Follow the deployment process for [deploying to Staging and Production]({{ site.baseurl }}/cloud/live/stage-prod-migrate.html). For Starter and Pro environments, you use the Project Web Interface to push code across environments.

Adobe recommends fully testing in the Staging environment before pushing to the Production environment. Make code changes in the Integration environment and begin the process to deploy across environments again.

<!-- link definitions -->

[addstorecode]: {{ site.baseurl }}/guides/v2.4/config-guide/multi-site/ms_websites.html#multi-storecode-baseurl
[config-multiweb]: {{ site.baseurl }}/guides/v2.4/config-guide/multi-site/ms_over.html
[Manage domains]: {{ site.baseurl }}cloud/cdn/configure-fastly-customize-cache.html#manage-domains
