---
title:  "Terraform Interpolation for DNS A Records"
date:   2016-04-14 07:00:00
categories: [terraform]
tags: [terraform]
---

Being a great fan of Terraform, I try to express and manage all components of my infrastructure as code. Some of the services and apps need DNS zone records configured on the internal DNS zone. In this article I will show you how to dynamically configure A records with Terraform.

For this example, let's say we want 3 App servers to start with. We want to take into consideration that the requirements might change in the future. We are going to make this number a variable.

```yaml
variable "app_cluster_count" {
    description = "Number of App instances we want in our cluster"
    default = 3
}
```

This is going to make it easy for everyone to change how many instances we want to launch. Just change the number and Terraform will take care of the rest.

I'm going to skip the instance creation part, as it's not relevant for this article.

For the rest of the article I'm going to use the AWS provider, but this can be easily done with PowerDNS as well.

Now let's configure the internal zone.

```yaml
resource "aws_route53_zone" "internal" {
    name = "myapp.internal"
    comment = "Internal zone for MyApp"
    vpc_id = "${aws_vpc.my_vpc.id}"
}
```

Now we can start adding our DNS records. Let's start with the A records. We want Terraform to configure our records regardless of how many instances we have. We are going to use variable interpolation:

```yaml
resource "aws_route53_record" "app_cluster" {
  count = "${var.app_cluster_count}"
  zone_id = "${aws_route53_zone.internal.zone_id}"
  name = "${format("app-cluster-%02d", count.index + 1)}"
  type = "A"
  ttl = "300"
  records = ["${element(aws_instance.app_cluster.*.private_ip, count.index)}"]
}
```

Let's dig a bit on what happens here.

First, we set a `count` equal to the value of `app_cluster_count`. This way Terraform will be able to keep track of which instance you are refering to.

We then want to set the A record name to something like `app-cluster-01.myapp.internal`, `app-cluster-02.myapp.internal`, etc. For this we use the `format` function which allows us to format strings. The syntax is the same as `sprintf` in Go. When we specified `count`, we said that we want 3 instances. This is how human logic works, we think in ordinal numbers. Terraform keeps track of the resources you made the same way arrays usually work. They are zero-index based. Because of this, `count.index` starts at `0`. We don't want our records to start at 0, so we add one to that value. This will not increment the value of count, it will just add one.

The last piece is `${element(aws_instance.app_cluster.*.private_ip, count.index)}`. Let's break it down.

* The `element` function takes as arguments a list and an index. Easy enough, we use our `count.index` as the index since we already know it will refer to right instance.

* Next, we want to get the value of the private IP address assigned to our instance. We could use `aws_instance.app_cluster.0.private_ip`, `aws_instance.app_cluster.1.private_ip`, etc. This is inefficient and not scalable. Every time we want to change the number of instances, we have to change the zone records configuration. In Terraform we can use what they call the **splat syntax**. Because we use `count` and `aws_instance.app_cluster.*.private_ip`, Terraform will know how many instances we have. It will replace `*` with the value of `count.index`. For each iteration it will create a `aws_route53_record` resource and increment the count.

We don't have explicit looping in Terraform, but we can use splat syntax can handle complex cases.

If you're interested in learning more about variable interpolation, check the [Terraform Interpolation Syntax][tf-interpolation] documentation page.

In the next article I'm going to show you something slightly more complex. We are going to pick up where we left and configure PTR records the same way.

Please check part 2 of this article: [Creating DNS PTR Records with Terraform][next-article]

Did you enjoy this article? Feel free to leave your comments or submit suggestions bellow.

[tf-interpolation]: https://www.terraform.io/docs/configuration/interpolation.html
[next-article]: https://liviu.io/2016/terraform-interpolation-dns-ptr-records/
