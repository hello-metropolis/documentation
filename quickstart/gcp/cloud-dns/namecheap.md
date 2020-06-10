# Metropolis
## Quickstart Guide

All domain registrars are a little different, but this guide will walk you through how to use _Namecheap_ to register a domain and hook it up to Google Cloud Platform's registrar.

I suggest using the `.host` domain as the cost of these domains are `$1.88` and it will allow you full control of the domain, and configure different sandbox environments with different domains.

**First**, search for the domain you're looking for on [Namecheap's Registration Page](https://www.namecheap.com/domains/registration).

**Second**, add it to your cart.

**Third**, register for an account and checkout with the domain.

**Fourth**, setup a CloudDNS zone.

* [Visit the page to create a Cloud DNS Zone](https://console.cloud.google.com/net-services/dns/zones).
* Click the button to `+ Create Zone`
* Fill out the form with these details:
  * `Zone type` keep public.
  * `Zone name` matches your domain name, replacing `.` with `-`.
  * `DNS name` needs to match your domain exactly.
  * Leave the rest of the defaults and press `Create`.

**Fifth**, point the domain to the Google Cloud DNS nameservers.

* Click the `Registrar setup` link.  It will show 4 domain nameservers you should point the domain to on Namecheap.  For me, they looked like: `ns-cloud-e1.googledomains.com.`
* [Visit the account dashboard](https://ap.www.namecheap.com/?) page and click the button to **Manage** the domain.
* Under the `NAMESERVERS` configuration, switch to `Custom DNS`.
* Enter the four nameservers you found on the Google interface.
* Press the green checkbox to save the changes in Namecheap.

**Sixth**, setup root domain to ensure it's registered successfully.

We've setup quite a few pieces and it could be helpful to test that these steps have worked as expected.

To ensure that it's working we can point the root domain to project.  The `quickstart` application has a `gh-pages` branch, so we can easily set it up to point to that page.

**On GitHub**

* Find the repo on GitHub's website
* Press the `Settings` button
* Under `GitHub Pages` find the option for `Custom domain`.
* Enter the domain you purchased and save the option.

**On GCP Cloud DNS**

* Find the zone file you just created.
* Press the `Add record set` and add the following 4 ip addresses
  * `185.199.108.153`
  * `185.199.109.153`
  * `185.199.110.153`
  * `185.199.111.153`
* Save the record

**Testing it**

After the DNS changes propagate when you visit the domain you purchased you will see a static page appear on the root URL of the domain.




https://console.cloud.google.com/net-services/dns/zones?project=hello-metropolis&dnsManagedZonessize=50
