---
title:  "Creating DNS PTR Records with Terraform"
date:   2016-04-19 07:00:00
categories: [terraform]
tags: [terraform]
---

In the [previous article][prev-article] we discussed how to create DNS A records for a variable number of instances. Today I'm going to show you how to create PTR records using Terraform.

PTR records are a bit more tricky. They are the reverse of an A record. A records point the hostname to an IP address, while PTR points an IP address to a hostname. This is also called reverse DNS resolution sometimes. If you want to learn more about DNS, [RFC1035][rfc1035] is a good place to start.

We already saw how Terraform's interpolation syntax works and how we can use to keep our code DRY. Due to the format of PTR records, we are going to have to use more of Terraform's functions. It might look intimidating at first, but once you will understand how it works it will make a lot more sense.

I'm going to assume that we are using `172.16.0.0/16` for our private subnet.

Let's first create our reverse zone.

```yaml
resource "aws_route53_zone" "reverse" {
    name = "16.172.in-addr.arpa"
    comment = "Reverse Internal for our Awesome App"
    vpc_id = "${aws_vpc.my_vpc.id}"
}
```

The above code is going to create the reverse zone for our `172.16.0.0/16` subnet. If you're not familiar with reverse DNS zones, please check the above link for RFC1035.

Now let's create the reverse DNS records. We are going to map the private IP address to an internal hostname for each instance.

```yaml
resource "aws_route53_record" "app_cluster_reverse" {
    count = "${var.app_cluster_count}"
    zone_id = "${aws_route53_zone.reverse.zone_id}"
    name = "${format("%s.%s.16.172.in-addr.arpa", element(split(".", element(aws_instance.app_cluster.*.private_ip, count.index)), 3), element(split(".", element(aws_instance.app_cluster.*.private_ip, count.index)), 2) )}"
    type = "PTR"
    ttl = "300"
    records = ["${format("app-cluster-%02d.myapp.internal", count.index + 1)}"]
}
```

We went into detail in the previous article for all options we have here. The only "ugly-looking" and "scary" thing is `${format("%s.%s.16.172.in-addr.arpa", element(split(".", element(aws_instance.app_cluster.*.private_ip, count.index)), 3), element(split(".", element(aws_instance.app_cluster.*.private_ip, count.index)), 2) )}`. This is the line that creates your PTR records, regardless how many instances you have.

To make it easier, let's break it into smaller pieces, starting from the end.

`element(aws_instance.app_cluster.*.private_ip, count.index)` - This is going to return us the private IP address of our instances, one at a time. Let's assume that the IP address of our first instance is `172.16.1.35`.

`split(".", element(aws_instance.app_cluster.*.private_ip, count.index))` - split() is a function that takes a delimiter and a string. It then splits the string into a list. In our case, the delimiter is `.` and the string `172.16.1.35`. At this stage the statement looks like this: `split(".", "172.16.1.35")`. The resulting list is going to have the elements `172`, `16`, `1`, and `35`.

`element(split(".", element(aws_instance.app_cluster.*.private_ip, count.index)), 3)` - This is the easiest one. `element(list, index)` is going to return the element found in a list at the specified index number. In human readable form this would be `element(["172", "16", "1", "35"], 3)`. Index numbers in a list start at `0`, so this is going to return `35`.

The second part, `element(split(".", element(aws_instance.app_cluster.*.private_ip, count.index)), 2)`, is exactly the same, the only difference is that it returns the 3rd element of the list, `1`.

The last piece we have to figure out, is `${format("%s.%s.16.172.in-addr.arpa", element(split(".", element(aws_instance.app_cluster.*.private_ip, count.index)), 3), element(split(".", element(aws_instance.app_cluster.*.private_ip, count.index)), 2) )}`.

We already know what that we get `35` and `1` out of the `element(split(element(...)))` functions. Given what we know, this `format` statement becomes `${format("%s.%s.16.172.in-addr.arpa", "35", "1")}`. We discussed in the previous article that `format` uses the same rules as `sprintf`. Each `%s` is replaced with a string, given as a parameter to the `format` function. Thus, Terraform will evaluate this to `35.1.16.172.in-addr.arpa`, which is exactly what we wanted to.

Terraform's interpolation syntax, although sometimes it looks complex, it's also powerful. I hope this makes more sense now. With practice it will become easier to use as well.

Did you enjoy this article? Feel free to leave your comments or submit suggestions bellow.

[prev-article]: https://liviu.io/2016/terraform-interpolation-for-dns-records/
[rfc1035]: https://tools.ietf.org/html/rfc1035
