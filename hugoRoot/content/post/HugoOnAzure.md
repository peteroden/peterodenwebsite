---
title: "Hugo on Azure"
date: 2018-04-05
description: "Easily run Hugo on Azure with deployment triggered by GitHub commits"
image: "/media/images/hugoOnAzure/gohugo.png"
categories: ["azure", "web"]
tags: ["hugo", "azure", "functions", "serverless", "static site", "JAMstack"]
draft: false
---

Late last year several other Global Black Belts and I decided to create a podcast series [https://www.azureninjas.cloud](https://www.azureninjas.cloud) and we were looking for a way to host it. We landed on using Hugo since this was mostly going to be static content and and several of us were already familiar with it.

I really wanted to minimize the infrastructure we needed to manage just to host the site, so I really wanted to build this all using serverless products in Azure. So I set out to figure out the best way to accomplish this. I wound up with a CDN sitting in front of blob storage. But this still meant that we had to manually deploy our content anytime there was content that changed. I really wanted to get a simple deployment pipeline, so I created an Azure Function that was triggered on GitHub commits of the source content.

We have been using this pattern for a while now and I have deployed several more sites using this same pattern. It has been working pretty well, so I thought I would generalize the project and share this for others to use. In this article I will walk through how I implemented this process.

If you just want to jump to the template you can get it from the GitHub repository at [https://github.com/codingwithsasquatch/hugoOnAzure](https://github.com/codingwithsasquatch/hugoOnAzure)

Now on to how it works. The workflow looks something like this:

![Hugo On Azure Architecture](/media/images/hugoOnAzure/hugoOnAzureArch.png "Hugo On Azure Architecture")

### Why not just use raw Azure Blob Storage?

So if you have implement a static site on another cloud like AWS, you may be asking why we don't just use raw Azure Blob Storage like you would with AWS S3? Well, Azure Storage is perfect for hosting the static content for incredibly cheap if you set the permissions such that the blobs are publicly readable, but there are a few things that keep you from simply using Azure Storage for the whole thing.

* First and probably the most painful item, is that there is no default document capability and requires all blobs to exist in a container ( basically a sub-directory). This is the top most requested feature for the storage team and has been for a couple years now, but it still isn't available. It's coming, but still not available yet. Feel free to pile on and vote for it [here](https://feedback.azure.com/forums/217298-storage/suggestions/6417741-static-website-hosting-in-azure-blob-storage).

* The other issue for my scenario is that there is no CI/CD with GitHub directly in Storage (This one applies to AWS S3 too). So you have to use something Like Jenkins, VSTS, Azure Automation, Functions or some other tool to push the file to storage when there is new content to be published. But I wanted this to be something that folks can use without having to set up any infrastructure. I wanted to have something that you could just deploy via an ARM template with a few configuration settings and have a working site.

## Here's how my template works:

### CDN

CDN can be used to help with the first point above. It can point to the container as the root for the site and with Azure Premium CDN, it can even use a rewrite rule to provide basic default document capability. One drawback to Azure CDN though is that you cannot set up the rewrite rules at deployment time nor and changes to the rules can take several hours to take effect. Not to mention Premium CDN is more expensive.

With this in mind we use Azure Standard CDN. Which allows us to push the files as close to the users as possible minimizing the delivery time to the users. Then we use functions for the URL rewriting. Using a CDN is optional and can be enabled or disabled when deploying the template.

### Function

Here comes Azure Functions to save the day, and nicely resolve the remaining items!

* First, Azure Functions Proxies enables me to address the default documents and allows me to do so in a simple programmatic manner as part of the deployment. While Azure Functions Proxies are pretty cool, but they still, unfortunately, lack regex which would be incredibly useful for URL rewriting. This means our proxies.json file quickly gets ugly. see the snippet below:

```json
{
    "$schema": "http://json.schemastore.org/proxies",
    "proxies": {
        "root": {
            "matchCondition": {
                "route": "/"
            },
            "backendUri": "https://%storageAccountName%.blob.core.windows.net/public/index.html"
        },"favicon": {
            "matchCondition": {
                "route": "/favicon.ico"
            },
            "backendUri": "https://%storageAccountName%.blob.core.windows.net/public/favicon.ico"
        },
        "level1": {
            "matchCondition": {
                "route": "/{level1}/"
            },
            "backendUri": "https://%storageAccountName%.blob.core.windows.net/public/{level1}/index.html"
        },
        "level2": {
            "matchCondition": {
                "route": "/{level1}/{level2}/"
            },
            "backendUri": "https://%storageAccountName%.blob.core.windows.net/public/{level1}/{level2}/index.html"
        },
        "level2Md": {
            "matchCondition": {
                "route": "/{level1}/{level2}/{name}.md"
            },
            "backendUri": "https://%storageAccountName%.blob.core.windows.net/public/{level1}/{level2}/{name}.md"
        }
    }
```

But I've already scripted the creation of this monstrosity for you so you don't have to worry about it. Oh and it is wicked fast and cheap.

* Second, since Azure Functions is built on top of the App Services platform, it brings many of the same features. Two such features are GitHub CI/CD deployment and kudu deployment scripts. I use the CI/CD functionality to automatically trigger a deployment to blob storage every time we receive a commit webhook from GitHub. I then use a custom kudu deployment script to build the static content and push them to the storage account.

* One additional feature is that I am able to host my backend APIs such as search on the same Azure Function using the same domain.

## How to use it

To use the template, clone or fork the [repository](https://github.com/codingwithsasquatch/hugoOnAzure). Then work with Hugo as you normally would in the hugoRoot directory. When you are ready to deploy it, make sure your changes are pushed to you guthub repo, and click the deploy to azure button in you repo on mine.

[![Deploy to Azure](http://azuredeploy.net/deploybutton.svg)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fcodingwithsasquatch%2FhugoOnAzure%2Fmaster%2Fazuredeploy.json)

You will be taken to a screen that looks something like this:

![Hugo On Azure Template Options](/media/images/hugoOnAzure/template.png "Hugo On Azure Template Options")

Fill in the highlighted fields and click purchase. your site will be deployed within a few minutes!



## Wrap Up

This is far from the only way to publish a static Hugo site on Azure, but it has been working extremely well and all for pennies a month. I hope not only this template is helpful, but also the walk-though of why I selected the components I did will help you with other scenarios. I'd love to hear any recommendations folks have for how this could be optimized. And If you have any improvements please send me a pull request!
