---
title: "Azure Event Generator"
date: 2018-03-30
description: "Event generation tool for all the Azure messaging services running entirely serverless"
image: "media/images/binary.jpg"
categories: ["Azure"]
tags: ["Azure", "Event Grid", "Event Hub", "Service Bus", "Queue", "Serverless", "Durable Function"]
draft: false
---

A few months back, several of my colleagues and I were putting together a workshop on how to design and run serverless applications on Azure. One scenario we wanted to demonstrate was processing real-time event streams. But we had an issue, we needed a source. Being that this was a workshop all about serverless, I set out to create a tool entirely serverless that generated events that we could then process. I call this tool Azure Event Generator. It is a pretty generic tool and is pretty useful for a a slew of applications, so I thought I would walk through what it is, how it works and how you can use it.

![picture of tool](/media/images/eventgen.png "Azure Event Generator")

## What it does

This tool allows you to select an Azure messaging service, provide connection parameters needed for your endpoint, then pick the message pattern, frequency, and duration you want for the messages. Then the service will send messages to your endpoint. This is great for testing performance, scaling, etc. or just learning how the various messaging services work.

## How it works

Azure Event Generator is built using Durable Functions on an Azure Function App. or those who haven't played with Durable Functions yet, this framework basically enables you to implement some very cool patterns like stateful services, fan out, fan in, singleton, etc. on top of Azure Functions. In this scenario I am basically just scratching the surface of what is possible with Durable Functions. Basically, it takes a provide connection string, frequency, and duration, and uses a durable function orchestrator's scheduler to send out batches of messages at a chosen interval for a period of time. This allows the function to easily scale out as needed but also means the function is only running when it is actively sending messages.

It has a SPA webpage that is served by same the Function App to provide a GUI to the underlying UI. I also built an abstraction for a messaging generator. This makes it easy for me to create custom message patterns. In our case, we were reading ninja battle parameters from a JSON file and using them to send events.

## How to use it

There are a few of ways you can use this:

* The easiest way is to just go to [https://aka.ms/eventgen](https://aka.ms/eventgen) and use the tool there pointed at your messaging endpoint.
* I have also published the source out on GitHub, so you can grab the code and deploy it on your own Azure subscription and customize it to your heart's content. The repo is [here](https://github.com/codingwithsasquatch/AzureEventGenerator).

## Feedback

As always, I welcome any feedback you have, especially pull requests and issues. I hope you find this tool useful.