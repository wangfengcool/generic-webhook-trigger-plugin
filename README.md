# Generic Webhook Trigger Plugin

[![Build Status](https://ci.jenkins.io/job/Plugins/job/generic-webhook-trigger-plugin/job/master/badge/icon)](https://ci.jenkins.io/job/Plugins/job/generic-webhook-trigger-plugin)

This is a Jenkins plugin that can:

 1. Receive any HTTP request, JENKINS_URL/generic-webhook-trigger/invoke
 2. Extract values

  * From POST content with [JSONPath](https://github.com/json-path/JsonPath) or [XPath](https://www.w3schools.com/xml/xpath_syntax.asp)
  * From the query parameters
  * From the headers

 3. Contribute those values as variables to the build

There is an optional feature to trigger jobs only if a supplied regular expression matches the extracted variables. Here is an example, let's say the post content looks like this:
```
{
  "before": "1848f1236ae15769e6b31e9c4477d8150b018453",
  "after": "5cab18338eaa83240ab86c7b775a9b27b51ef11d",
  "ref": "refs/heads/develop"
}
```

Then you can have a variable, resolved from post content, named `reference` of type `JSONPath` and with expression like `$.ref` .

The optional filter text can be set to `$reference` and the filter regexp set to [^(refs/heads/develop|refs/heads/feature/.+)$﻿](https://jex.im/regulex/#!embed=false&flags=&re=%5E(refs%2Fheads%2Fdevelop%7Crefs%2Fheads%2Ffeature%2F.%2B)%24) to trigger builds only for develop and feature-branches.

## Use case

This means it can trigger on any webhook, like:
* [Bitbucket Cloud](https://confluence.atlassian.com/bitbucket/manage-webhooks-735643732.html)
* [Bitbucket Server](https://confluence.atlassian.com/bitbucketserver/managing-webhooks-in-bitbucket-server-938025878.html)
* [GitHub](https://developer.github.com/webhooks/)
* [GitLab](https://docs.gitlab.com/ce/user/project/integrations/webhooks.html)
* [Gogs](https://gogs.io/docs/features/webhook) and [Gitea](https://gitea.io/)
* [Assembla](https://blog.assembla.com/AssemblaBlog/tabid/12618/bid/107614/Assembla-Bigplans-Integration-How-To.aspx)
* And many many more!

The original use case was to build merge/pull requests. You may use the Git Plugin as described in [this blog post](http://bjurr.com/continuous-integration-with-gitlab-and-jenkins/) to do that. There is also an example of this on the [Violation Comments to GitLab Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Violation+Comments+to+GitLab+Plugin) page.

You may want to report back to the invoking system. [HTTP Request Plugin](https://wiki.jenkins-ci.org/display/JENKINS/HTTP+Request+Plugin) is a very convenient plugin for that. 

If a node is selected, then all leafs in that node will be contributed. If a leaf is selected, then only that leaf will be contributed.

There are websites to help fiddle with the expressions. You may want to checkout:

* [This website](https://jsonpath.curiousconcept.com/) to fiddle with JSONPath.
* [This website](http://www.freeformatter.com/xpath-tester.html) to fiddle with XPath.
* [This website](https://jex.im/regulex/) to fiddle with regexp.

When using the plugin in several jobs, you will have the same URL trigger all jobs. If you want to trigger only a certain job you can:

* Use the `token`-parameter and have different tokens for different jobs.
* Add some request parameter (or header, or post content) and use the **regexp filter** to trigger only if that parameter has a specific value.

Available in Jenkins [here](https://wiki.jenkins-ci.org/display/JENKINS/Generic+Webhook+Trigger+Plugin).

## Authentication

There is a special `token` parameter. When supplied, it is used with [BuildAuthorizationToken](http://javadoc.jenkins-ci.org/hudson/model/BuildAuthorizationToken.html) to authenticate.

It might be a good idea to have a different token for each job. Then only that job will be visible for that request. This will increase performance and reduce responses of each invocation.

![Parameter](https://github.com/jenkinsci/generic-webhook-trigger-plugin/blob/master/sandbox/configure-token.png)

The token can be supplied as a:

* Request parameter: `curl -vs http://localhost:8080/jenkins/generic-webhook-trigger/invoke?token=abc123 2>&1`
* Token header: `curl -vs -H "token: abc123" http://localhost:8080/jenkins/generic-webhook-trigger/invoke 2>&1`
* *Authorization* header of type *Bearer* : `curl -vs -H "Authorization: Bearer abc123" http://localhost:8080/jenkins/generic-webhook-trigger/invoke 2>&1`

## Troubleshooting

It's probably easiest to do with curl. Given that you have configured a Jenkins job to trigger on Generic Webhook, here are some examples of how to start the jobs.

```
curl -vs http://localhost:8080/generic-webhook-trigger/invoke 2>&1
```

This may start your job, if you have enabled "**Allow anonymous read access**" in global security config. If it does not, check the Jenkins log. It may say something like this.

```
INFO: Did not find any jobs to trigger! The user invoking /generic-webhook-trigger/invoke must have read permission to any jobs that should be triggered.
```

And to authenticate in the request you may try this.

```
curl -vs http://username:password@localhost:8080/generic-webhook-trigger/invoke 2>&1
```

If you want to trigger with some post content, curl can dot that like this.
```
curl -v -H "Content-Type: application/json" -X POST -d '{ "app":{ "name":"GitHub API", "url":"http://developer.github.com/v3/oauth/" }}' http://localhost:8080/jenkins/generic-webhook-trigger/invoke?token=sometoken
```

## Screenshots

![Generic trigger](https://github.com/jenkinsci/generic-webhook-trigger-plugin/blob/master/sandbox/generic-trigger.png)

If you need the resolved values in pre build steps, like git clone, you need to add a parameter with the same name as the variable.

![Parameter](https://github.com/jenkinsci/generic-webhook-trigger-plugin/blob/master/sandbox/parameter-git-repo.png)

## Job DSL Plugin

This plugin can be used with the Job DSL Plugin. There is also an example int he [Violation Comments to GitLab Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Violation+Comments+to+GitLab+Plugin) wiki page.

```
job('Generic Job Example') {
 parameters {
  stringParam('VARIABLE_FROM_POST', '')
 }

 triggers {
  genericTrigger {
   genericVariables {
    genericVariable {
     key("VARIABLE_FROM_POST")
     value("\$.something")
     expressionType("JSONPath") //Optional, defaults to JSONPath
     regexpFilter("") //Optional, defaults to empty string
     defaultValue("") //Optional, defaults to empty string
    }
   }
   genericRequestVariables {
    genericRequestVariable {
     key("requestParameterName")
     regexpFilter("")
    }
   }
   genericHeaderVariables {
    genericHeaderVariable {
     key("requestHeaderName")
     regexpFilter("")
    }
   }
   printContributedVariables(true)
   printPostContent(true)
   regexpFilterText("\$VARIABLE_FROM_POST")
   regexpFilterExpression("aRegExp")
  }
 }

 steps {
  shell('''
echo $VARIABLE_FROM_POST
echo $requestParameterName
echo $requestHeaderName
  ''')
 }
}
```

## Pipeline Multibranch

This plugin can be used with the [Pipeline Multibranch Plugin](https://jenkins.io/doc/pipeline/steps/workflow-multibranch/#properties-set-job-properties). Here is an example:

With a Jenkinsfile like this:

```
node {
 properties([
  pipelineTriggers([
   [$class: 'GenericTrigger',
    genericVariables: [
     [key: 'reference', value: '$.ref'],
     [
      key: 'before',
      value: '$.before',
      expressionType: 'JSONPath', //Optional, defaults to JSONPath
      regexpFilter: '', //Optional, defaults to empty string
      defaultValue: '' //Optional, defaults to empty string
     ]
    ],
    genericRequestVariables: [
     [key: 'requestWithNumber', regexpFilter: '[^0-9]'],
     [key: 'requestWithString', regexpFilter: '']
    ],
    genericHeaderVariables: [
     [key: 'headerWithNumber', regexpFilter: '[^0-9]'],
     [key: 'headerWithString', regexpFilter: '']
    ],
    printContributedVariables: true,
    printPostContent: true,
    regexpFilterText: '',
    regexpFilterExpression: ''
   ]
  ])
 ])

 stage("build") {
  sh '''
  echo Variables from shell:
  echo reference $reference
  echo before $before
  echo requestWithNumber $requestWithNumber
  echo requestWithString $requestWithString
  echo headerWithNumber $headerWithNumber
  echo headerWithString $headerWithString
  '''
 }
}
```

It can be triggered with something like:

```
curl -X POST -H "Content-Type: application/json" -H "headerWithNumber: nbr123" -H "headerWithString: a b c" -d '{ "before": "1848f12", "after": "5cab1", "ref": "refs/heads/develop" }' -vs http://admin:admin@localhost:8080/jenkins/generic-webhook-trigger/invoke?requestWithNumber=nbr%20123\&requestWithString=a%20string
```

And the job will have this in the log:

```
Contributing variables:

    headerWithString_0 = a b c
    requestWithNumber_0 = 123
    reference = refs/heads/develop
    headerWithNumber = 123
    requestWithNumber = 123
    before = 1848f12
    requestWithString_0 = a string
    headerWithNumber_0 = 123
    headerWithString = a b c
    requestWithString = a string
```


## Plugin development
More details on Jenkins plugin development is available [here](https://wiki.jenkins-ci.org/display/JENKINS/Plugin+tutorial).

A release is created like this. You need to clone from jenkinsci-repo, with https and have username/password in settings.xml.
```
mvn release:prepare release:perform
```
