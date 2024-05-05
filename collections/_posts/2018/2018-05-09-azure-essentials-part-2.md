---
layout: post
title: "Azure Essentials: publishing an ASP.NET Core 2 MVC application"
categories: blog
---

In this post about the very basics of Microsoft Azure, I'll show how to publish an ASP.NET Core 2 MVC application to _the cloud_. We'll be using Visual Studio for this.

## Creating an App Service plan

The first thing we'll need is an App Service plan. This is essentially the hosting plan for your web applications. It determines the capabilities of the underlying hardware your applications will be running on, as well as the region where everything will be hosted. You can have multiple applications running on the same plan.

To create a new App Service plan, open the App Service plans service:

![](/assets/img/blog/2018/05/app-service-plan-service.png)

From there, select "add" to start creating a new App Service plan:

![](/assets/img/blog/2018/05/app-service-plan-add.png)

The blade for creating a new App Service plan will open:

![](/assets/img/blog/2018/05/app-service-plan-create.png)

The only really important thing to consider here is the pricing tier. There's a myriad of features to consider, which I won't get in to but are explained by Microsoft [here](https://azure.microsoft.com/en-us/pricing/details/app-service/plans/). For most development purposes, the Basic B1 plan will cover everything you'll need. For very simple applications, the F1 or D1 shared infrastructure plans are an even cheaper option.

Once you've filled out all the information, hit "create" to have Azure create the App Service plan for you.

## Creating a Web App

Now that our App Service plan is in place, we can create a Web App to host our ASP.NET application in.

To create a new Web App, open the App Services service:

![](/assets/img/blog/2018/05/app-services-service.png)

From there, click "add" to start creating a new App Service:

![](/assets/img/blog/2018/05/app-service-add.png)

A whole bunch of options will present themselves. Select the one that says "Web App":

![](/assets/img/blog/2018/05/web-app-add.png)

This will open an informational blade that just contains a "create" button. Click it! Click the button!

![](/assets/img/blog/2018/05/web-app-create.png)

This will _finally_ open the Web App creation blade:

![](/assets/img/blog/2018/05/web-app-create-2.png)

You'll enter the public address of your Web App here, as well as the underlying OS it'll be running on. You'll also select the App Serivce plan this Web App will use. In this case, we can select the plan we've created in the previous step.

Finally, there's the option of enabling Application Insights. Strangely enough, if you select "on" here you can only specify a region for AppInsights to use. If you proceed, I've found that Azure will then create a new AppInsights resource for you in that region, and link your new Web App to it. However, chances are you've already got an existing AppInsights instance running, and you don't want to create a new one for each Web App you create. In that case, you should select "off" here, and manually configure your Web App later to use the existing AppInsights resource:

![](/assets/img/blog/2018/05/web-app-appinsights.png)

## Deploying the application

We are now ready to deploy our MVC application. I'm assuming you've already got one. If you don't, no worries: even the MVC template that comes with Visual Studio should work, unaltered. You shouldn't really have to do anything special, just because your application will end up running in Azure.

So, anyway, about that deployment. There's multiple ways of going about it, but since we're using Visual Studio, the easiest way is to import the publish profile from Azure. From your Web App's overview page, simply select "Get publish profile":

![](/assets/img/blog/2018/05/web-app-publish-profile.png)

This will download a "your-web-app.PublishSetting" file, which we can then import in Visual Studio:

![](/assets/img/blog/2018/05/web-app-publish-import.png)

Careful: I've found that once you import the publish profile, Visual Studio starts publishing right away.

Assuming everything goes right, Visual Studio will automatically open your Web App in the browser as soon as publishing has finished. Congratulations. You now have a web application running in _the cloud_.

### About that 502.5 error...

A word of caution regarding ASP.NET Core 2 and the Azure 502.5 error: there's been numerous reports of Web Apps running into this error after updating to the latest version of the Microsoft.AspNetCore.All package. The version number doesn't seem to matter, just that it's a recent release. I ran into this problem with version 2.0.8 (which was only released days ago), but people have also had issues with earlier versions. In my case, downgrading to 2.0.7 fixed the problem. It seems Azure needs time to "catch up" to the latest Core developments, or something. So if your MVC application runs in Azure, this is something to keep in mind.
