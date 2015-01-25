---
layout: post
title:  "Paperclip - Swapping S3 with a drop-in replacement called nibbler.io"
date:   2015-01-01 13:51:02
categories: paperclip aws s3 nibbler.io
---

I recently had to migrate one of our customers from [Amazon S3](http://aws.amazon.com/s3/) to [Nibbler](http://acorn.nibbler.io/).
The application works on Ruby on Rails and uses one of the popular gems called [Paperclip](https://github.com/thoughtbot/paperclip/) for dealing with attachments.

Nibbler advertises itself as a drop-in S3 replacement and I was hopeful that it would mean only a simple configuration change and it was indeed all that it took.
However, I ended up reading Paperclip's code to figure it all out. I hope that this post would save you some time if you are stuck in a similar situation.

Here's what our original paperclip configuration looked like:

{% highlight ruby %}
config.paperclip_defaults = {
  storage: :s3,
  s3_credentials: {
    bucket: "bucketname",
    access_key_id: "4ELJLET5IAK4MWA4DHIQ",
    secret_access_key: "bSRsAP0nAJU+0pFqKMN09t/MMoxh9CU/JqXeYF"
  },
  s3_protocol: 'https'
}
{% endhighlight %}

### Challenge 1: Mapping the terms
While S3 uses the terms buckets and objects, Nibbler calls them bunches and acorns
The access key id and secret are named similar in both.
So, this part was easy - I simply replaced the s3_credentials in the configuration.

### Challenge 2: Changing the endpoint to nibbler's url
There is a configuration parameter available under s3_credentials named s3_host_name which makes paperclip start pointing to a custom url. Here - almond.nibbler.io

### Challenge 3: Nibbler does not support virtual hosted style urls
Say your bucket name is "johnsmith.eu", you are able to access the objects using a url that looks like

```
http://johnsmith.eu.s3-eu-west-1.amazonaws.com/homepage.html
```

More info related to Virtual hosting in S3 can be found
[here](http://docs.aws.amazon.com/AmazonS3/latest/dev/VirtualHosting.html).

This is what paperclip's S3 adapter chooses generally and unfortunately nibbler does support it

S3 also supports path-style urls like

```
http://s3-eu-west-1.amazonaws.com/johnsmith.eu/homepage.html
```

Paperclip uses the path-style urls only in certain conditions defined in

```
AWS::S3::Client::Validator#dns_compatible_bucket_name?
```
{% highlight ruby %}
# Returns true if the given `bucket_name` is DNS compatible.
#
# DNS compatible bucket names may be accessed like:
#
#     http://dns.compat.bucket.name.s3.amazonaws.com/
#
# Whereas non-dns compatible bucket names must place the bucket
# name in the url path, like:
#
#     http://s3.amazonaws.com/dns_incompat_bucket_name/
#
# @return [Boolean] Returns true if the given bucket name may be
#   is dns compatible.
#   this bucket n
#
def dns_compatible_bucket_name?(bucket_name)
  return false if
    !valid_bucket_name?(bucket_name) or

    # Bucket names should be between 3 and 63 characters long
    bucket_name.size > 63 or

    # Bucket names must only contain lowercase letters, numbers, dots, and dashes
    # and must start and end with a lowercase letter or a number
    bucket_name !~ /^[a-z0-9][a-z0-9.-]+[a-z0-9]$/ or

    # Bucket names should not be formatted like an IP address (e.g., 192.168.5.4)
    bucket_name =~ /(\d+\.){3}\d+/ or

    # Bucket names cannot contain two, adjacent periods
    bucket_name['..'] or

    # Bucket names cannot contain dashes next to periods
    # (e.g., "my-.bucket.com" and "my.-bucket" are invalid)
    (bucket_name['-.'] || bucket_name['.-'])

  true
end
{% endhighlight %}

So, I had to define the nibbler "bunch" name such that it meets one of the conditions that makes the method return false.
This was the only hard part to figure out since it wasn't defined as a configuration option.


#### Here's how the configuration finally looked like at the end of the exercise:

{% highlight ruby %}
config.paperclip_defaults = {
  storage: :s3,
  s3_credentials: {
    bucket: "my-very-long-bucket-name-wBFG7ChkX3L1JGDG0mXupVlY4QlWbLPGJhtSmeUJNOBrIV6pE1S77x5ougOQOpb",
    access_key_id: "4ELJLET5IAK4MWA4DHIQ",
    secret_access_key: "bSRsAP0nAJU+0pFqKMN09t/MMoxh9CU/JqXeYF"
    s3_host_name: "almond.nibbler.io"
  },
  s3_protocol: 'https'
}
{% endhighlight %}