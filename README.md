# jenkins_bitbucket_webhook_integration

### Overview

jenkins-Bitbucket integration via webhooks which help you to trigger an automatic jenkins job via HTTP(S) event payload ( JSON/XML request)

Bitbucket has an inbuilt webhook mechanisim https://confluence.atlassian.com/bitbucketserver/managing-webhooks-in-bitbucket-server-938025878.html This can be configured to send webhook request for a various events from a repository


### How it works ?

![jenkins_bitbucket integration](https://user-images.githubusercontent.com/28682822/83739803-f4ebad00-a64d-11ea-9642-8e15637b6f73.png)


### Requirments

1. Jenkins plugins
     a. Generic webhook trigger plugin - https://plugins.jenkins.io/generic-webhook-trigger/ - this plugin allows any webhooks to trigger a build in jenkins with variables               contributed from from JSON/XML.
     b. Stash Notifier Plugin - https://wiki.jenkins.io/pages/viewpage.action?pageId=66453548 - this plugin notifies bitbucket about the status of the jenkins build

2. Bitbucket - webhooks need to be configured


### Bitbucket webhook Configuration

To configure webhook - navigate to Bitbucket repositories settings -> webhook

update the name of the webhook and jenkins url with appropriate path 

`https(s)://<yourjenkins.url>/generic-webhook-trigger/invoke`

And select events which your tigger need to be invoked , The payload request will be sent via JSON https://confluence.atlassian.com/bitbucket/event-payloads-740262817.html

![bitbucket update](https://user-images.githubusercontent.com/28682822/83875412-f3de7c80-a72e-11ea-8e52-dfbdac144a81.PNG)


## Jenkins Pipeline Configuration Setting

### Manual Job configuration

1. To configue job with generic webhook trigger plugin - navigate to the job configuration and simply add it as a trigger to your job

![generic webhook trigger](https://user-images.githubusercontent.com/28682822/83877594-b1b73a00-a732-11ea-95e1-94f679f1fb74.PNG)

you can update the genric parameters for the job
Extract values
 - from `post` content with `JSONpath` or `Xpath`
 - from the query parameters
 - from the headers
 
 Eg below i have filtered the post content parameters (gitbranch) from the bitbucket webhook request
 
 ![post_content](https://user-images.githubusercontent.com/28682822/83906525-a298b200-a75b-11ea-84c4-e67bbad3a4b3.PNG)
 
 When using the plugin in several jobs, you will have the same url trigger for all jobs. If you want to trigger only a certain job you can:
 
 - use the `token` parameter have diffrent tokens for diffrent jobs. Using only the token means only jobs with that exact token will be    visible for that request. This will increase performance and reduce responses of each ivocation.
 - or , add some request parameter (or header,or post content) and use the regexp filter to trigger only if that parameter has a            specific value

![token](https://user-images.githubusercontent.com/28682822/83906918-5437e300-a75c-11ea-8756-137a1e73872b.PNG)

if you configure token here,you can mention the same in bitbucket webhhook setting as below to direct the trigger to particular job.

`http(s)://<jenkins_url>/generic-webhook-trigger/invoke?token=testing`

you can mention the cause of the job - so whenever the job trigggered it displayed with the cause

![cause](https://user-images.githubusercontent.com/28682822/83907923-1d62cc80-a75e-11ea-9aac-42860ce4bc6c.PNG)

### jenkins Pipeline job DSL sample

```
pipelineJob('PR automation job') {
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
	causeString ('***** PR created by $Commiter for repo*******')
	token ('testing')
	printpostcontent(true)
	}
  }

 definition {
  cps {
   // Or just refer to a Jenkinsfile containing the pipeline
   script('''
    node {
     stage('Some Stage') {
      println "VARIABLE_FROM_POST: " + VARIABLE_FROM_POST
     }
    }
   ''')
   sandbox()
  }
 }
} 
```
#### Notifybitbucket function
```
try { 
  
   ~~ code
   this.notifyBitbucket('SUCCESS`)
   } catch {
   this.notifyBitbucket('FAILED')
   }
   
  def notifyBitbucet(string state) {
	def gitcommitID = ("${commitID}" == null) ? "$(env.GIT_COMMIT)" : "${commitID}"
	if ('SUCCESS' == state || 'FAILED' == state) {
	   currentBuild.result = state
	}
    notifyBitbucket(
            commitSha1: '${gitcommitID}',
            credentialsId: '${gitCreds}',
            disableInprogressNotification: false,
            considerUnstableAsSuccess: true,
            ignoreUnverifiedSSLPeer: true,
            includeBuildNumberInKey: false,
            prependParentProjectKey: false,
            projectKey: '',
            stashServerBaseUrl: 'https://<stash url>'
			)
		}
  ```
  
  Once you have done all the required configuration - " PR automation will and notify the PR with results"
  
  ## Terraform Compliance
  
IAC evolving into developers world globally, to maintain and manage the code into production environment is important aspect the work above will give you the oppurtunity to unit test your code automatically with the help of below utilities

- https://github.com/eerkunt/terraform-compliance (BDD testing )
- https://github.com/open-policy-agent/conftest ( Policy test against code )

:) Happy coding 
