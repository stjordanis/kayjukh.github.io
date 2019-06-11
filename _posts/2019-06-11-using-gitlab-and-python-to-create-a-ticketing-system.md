---
title: Using Python and Gitlab to create a ticketing system
header:
  overlay_image: /assets/images/sebastian-unrau-47679-unsplash.jpg
categories: [programming]
tags: [python, gitlab, gitlab premium, rest api, ticketing system, service desk]
classes: wide
author_profile: true
---

When you are working with a growing team and you are the one in charge of all the IT-related requests (buying hardware, creating accounts, etc.), there comes a time when simply using emails or direct messages to handle those request becomes untractable. You may forget a message that was sent to you while you were busy doing something else, or you might get a lot of requests, some of which you might overlook. This is usually the best opportunity to have a look at ticketing systems.

A ticketing system is a system that centralizes all your customers and/or colleagues requests and helps you keep track of which ones were already processed, which ones need more time, etc. Usually, there is also a way for you and the person making a request to chat about the details of what should be done. There are several great ticketing system providers, including Zendesk and Freshdesk for example. But what if you want to avoid having to setup one of those additional systems and use something that you are more familiar with? In this post, we will build a simple ticketing system by using Gitlab Service Desk, the Gitlab REST API and Python.

**Warning:** This post assumes that you have an instance of Gitlab Premium readily available.
{: .notice--warning}

# Setting up a new Gitlab repository

The first step is to setup a new blank Gitlab repository for your ticketing system. I called mine `ticketing`, but you can give it any name.
![Create a new blank project](/assets/images/2019-06-11-using-gitlab-and-python-to-create-a-ticketing-system/new_project.png)

As with any repository, you can give it a description, choose to make it public, internal or private and you can initialize it with a README file. I will keep the default settings for the sake of demonstration.

# Enabling Service Desk

The most important feature that we will be using in this tutorial is the Gitlab Service Desk. Service Desk is a feature that was released in Gitlab Premium v9.1. It is aimed at easing the interaction between a development team and customers. Quoting the [official release annoucement](https://about.gitlab.com/2017/04/22/gitlab-9-1-released/),

> Your customers or anyone in contact with the people in your project can email bugs, feature requests, or any other general feedback directly into your GitLab project. In turn, any GitLab users can respond straight from the project.

We can take advantage of this feature and use it to send support tickets directly to our new repository via email.

To enable Service Desk, simply go to the repository's general settings and scroll down to **Service Desk**. Clock *Expand* and enable it (it may already be enabled by default). You will see that Gitlab automatically generated an email address for the support requests to be sent to.
![Enabling Service Desk](/assets/images/2019-06-11-using-gitlab-and-python-to-create-a-ticketing-system/enable_service_desk.png)

Now that everything is set up, we can start scripting our very own ticketing system!

# A ticketing system in Python

We will use the Gitlab REST API (documentation available [here](https://docs.gitlab.com/ee/api/)) to interact with our ticketing system repository, along with the Python `requests` library. To install the library, simply follow the [official installation instructions](https://2.python-requests.org/en/master/user/install/#install). The following steps were all tested with Python 3.7.3 and `requests` 2.22.0.

Let us create a Python script and name it `gitlab-tickets.py`. The first thing to do is import the content of the `requests` library.
```python
import requests
```

Most of the API functionalities require some form of authentication. You can either use OAUTH tokens or use a personal access token. I chose the latter solution. The instructions to create a personal access token are available in [the Gitlab documentation](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html).

The general form of a Gitlab API request is as follows:
```
https://gitlab.com/api/v/projects/<projectId>/<request>?private_token=<privateToken>
```
You can get your projects ID on your project repository's home page.
![Finding your project's ID](/assets/images/2019-06-11-using-gitlab-and-python-to-create-a-ticketing-system/project_id.png)

To make the code more readable, let us add the following lines in the Python script:
```python
privateToken = 'your-private-token'
privateTokenStr = 'private_token={}'.format(privateToken)
projectId = 12345678
prefix = 'https://gitlab.com/api/v4/projects/{}'.format(projectId)
```
Make sure to replace `privateToken` and `projectId` with your personal access token and your project's ID respectively.

To test our setup, let us add
```python
print(requests.get('{}?{}'.format(prefix, privateTokenStr)).json())
```
to the script. This will try to retrieve the project properties from Gitlab and print them in JSON format. You can run the script by simply executing
```sh
python gitlab-tickets.py
```
You should see a bunch of JSON printed out to the screen. If not, make sure that your personal access token as well as your project ID are correct. You can now safely remove the last line we added to the Python script. Next up, we will plan some of the features of our ticketing system.

## Planning some features

At this point, everything is working as expected and you should be able to make requests to your repository using the Gitlab API. We are ready to start scripting our ticketing system.

To keep things simple, we will only incorporate a few features to our ticketing system. The support tickets will be managed in the repository's issues and will be labeled according to categories that we will define. Assignation of labels should be automated, and will be based on a common issue title prefix. The script should be the one responsible for adding the missing labels.

## Implementation

Now that we have an idea of what we are aiming for, let us dive into the code.

First, we define some ticket categories. We will use a Python dictionary to hold the data. Let us add two categories, namely *Hardware* for hardware-related stuff, and *Gitlab Account* for Gitlab account creation requests.
```python
categories = [
	{
		"id": "Hardware"
		"label": {
			"name": "hardware",
			"color": "#f45562"
		}
	},
	{
		"id": "Gitlab Account",
		"label": {
			"name": "gitlab-account",
			"color": "#3450f6"
		}
	}
]
```
Each category has a unique identifier as well as an associated label, with a name and a color.

Next, we will have to add those labels to the repository. To do that, we use [the `labels` request](https://docs.gitlab.com/ee/api/labels.html#create-a-new-label) provided by the Gitlab API. If no error occurs, the request will return the status code `201`. If a label already exists, we simply ignore the error and continue the execution.
```python
for category in categories:
	label = category.get('label')
	resp = requests.post('{}/labels?{}'.format(prefix, privateTokenStr), json = label)
	if resp.status_code != 201:
		if resp.status_code != 409: # Label already exists
			raise Exception('POST /labels/: {} - {}'.format(resp.status_code, resp.text))
```

We then retrieve the list of all the issues from the repository. This will allow us to iterate over them and automatically assign the right label to each individual issue, based on its title.
```python
resp = requests.get('{}/issues?{}'.format(prefix, privateTokenStr))
if resp.status_code != 201:
		raise Exception('GET /issues/: {} - {}'.format(resp.status_code, resp.text))
```

If there is a single issue, `resp.json()` will return a dictionary, whereas if there are multiple issues, it will return a list of dictionaries. Let us normalize this and always use a list.
```python
jsonResp = resp.json()
if not isinstance(jsonResp, list):
	jsonResp = [jsonResp]
```

Next, we iterate over all the issues and search for the ones posted by Service Desk. We then check if any of the predefined title headers are present. A typical service ticket title will look like
```
Service Desk (from john.doe@mail.com): [Hardware] Buying a new screen
```
We try to find a substring of this title corresponding to one of the category IDs inside square brackets and attach the corresponding label to it.
```python
for issue in jsonResp:
    title = issue.get('title')
    serviceDeskStr = 'Service Desk'
    if serviceDeskStr in title:
        for category in categories:
            tag = '[{}]'.format(category.get('id'))
            if tag in title:
                label = category.get('label')
                issueIid = issue.get('iid')
                labelName = label.get('name')
                assignLabels = issue.get('labels')
                if not labelName in assignLabels:
                    assignLabels.append(label.get('name'))
                    updateData = { "labels": assignLabels }
                    resp = requests.put('{}/issues/{}?{}'.format(prefix,
					issueIid, privateTokenStr), json=updateData)
                    if resp.status_code != 200:
                        raise Exception('PUT /issues/{}/: {} - {}'.format(issueIid,
						resp.status_code, resp.text))
```

That's it! We now have a simple ticketing system entirely integrated into Gitlab. Each time you run this script, all the issues will be updated with the right labels automatically. Let us test it out.

## Testing it out

Let us write an email to the address that was generated by Gitlab for our repository. I will request for a new screen:
```
Subject: [Hardware] Buying a new screen
Message:
	Hi!

	I just wanted to ask if it would be possible to get a new screen for my office? The one I currently use is a little to small.

	Thanks,
	Jean-Michel.
```
Note that I use one of the category IDs inside square brackets in the subject of my email. This is very important, as it will allow our script to give the right label to the resulting Gitlab issue.

The new issue is now visible on the repository.
![Newly created Service Desk issue](/assets/images/2019-06-11-using-gitlab-and-python-to-create-a-ticketing-system/new_issue.png)
and we can respond to it like any other issue. The answer will automatically be forwarded to the mailbox of the person who posted the request.
![Answering a ticket](/assets/images/2019-06-11-using-gitlab-and-python-to-create-a-ticketing-system/new_issue_detail.png)

# Concluding thoughts

As we have seen in this post, it is very easy to setup a simple ticketing system using Python and Gitlab Service Desk. To fully automate the process, you could simply add the Python script we wrote to a CI task, triggering it regularly to update all newly created issues.
