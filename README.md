## Description

This is a tool to test F5 LTM iRule logic

## Getting Started

**Example normal test:**

        ./test.sh -s www.acme.com

**Example normal output:**

                Testing www.acme.com

        W - 0001 - logic not currently testable
        P - 0002 - http://www.acme.com/evilpage.html
                discarded request as expected
        P - 0003 - http://www.acme.com/original/location1
                redirects to http://www.24hourfitness.com/new/location1 as expected
        P - 0004 - http://www.acme.com/original/location2
                redirects to http://www.24hourfitness.com/new/location2 as expected
        P - 0005 - http://www.acme.com/index.php
                selected pool acme_prd_pool as expected
        P - 0006 - http://www.acme.com/index.html
                selected pool acme_prd_pool as expected

        Pass: 6, Warn: 1, Fail: 0, Total: 7

        Test took 5 seconds

**Example tap test:**

        ./test.sh -s www.acme.com -o tap

**Example tap output:**

                Testing www.acme.com

        not ok 0001 - #SKIP logic not currently testable
        ok 0002 - http://www.acme.com/evilpage.html -
        ok 0003 - http://www.acme.com/original/location1 -
        ok 0004 - http://www.acme.com/original/location2 -
        ok 0005 - http://www.acme.com/index.php -
        ok 0006 - http://www.acme.com/index.html -

        Pass: 6, Warn: 1, Fail: 0, Total: 7

        Test took 5 seconds

Additional testing can be performed by passing the -e option to the testing 
script.  This will check for an ASM block page, specific sorry content, and 
check for multiple redirect responses for a given request.  Please read the 
irule-tester library file for more details.

Debugging can be enabled by passing the -d option.  This will enable Bash 
debugging and is VERY verbose.  Be warned...

## Supported Platforms

This code was developed and tested using CentOS 5, but is assumed to work
on other platforms as well.

## Dependencies

* bash >= 3.2.25
* curl >= 7.15.5
* dos2unix >= 3.1

## Overview

This utility relies on two required components and has an optional third.  

1. The irule-tester library (./lib/irule-tester - required)
2. A test.sh file the contains your test cases (./test.sh - required)
3. Adding the http-response.irule to the site you want to test if you are not
   using cookie persistence (./support/http-response.irule - optional)

### - irule-tester library - 

This is a bash script that contains default variables and the required 
functions to make this whole thing come together.  The file is well documented 
and contains sane defaults, but you may need to adjust the settings to fit 
your environment.

### - test.sh -

This is the file that contains your test cases.  The file can be named 
anything you want.  I recommend having a seperate file per site that 
you want to test.

**Usage:**

	./test.sh -s target_site [ -o (plain|tap) ] [-e] [-d]

**Example:**

	./test.sh -s www.acme.com

#### Test methods

The following testing methods are currently available.  Additional methods 
will be added as they are required.

`cannot_test` 

  * Use this method when you need to put a placeholder in your test cases for iRule logic that cannot currently be tested due to the way it was written

**Usage:**

  cannot\_test TEST\_CASE\_NUMBER [TEST\_DESCRIPTION]

**Example:**

	cannot_test 0001 "Description of test"

---

`header_should_contain`

  + Use this method when you need to validate that the response to a given request contains a specific string in the header 

**Usage:**

  header\_should\_contain TEST\_CASE\_NUMBER PARAMETERIZED\_URL MATCH\_STRING

**Example:**

        header_should_contain 0001 http://${targetSite}/index.html acmeTestHeader

        Note: the $targetSite variable is replaced by the value passed to test.sh with the -s switch

---

`should_discard`

  + Use this method when you need to validate that a given request is ignored or discarded by the F5

**Usage:**

  should\_discard TEST\_CASE\_NUMBER PARAMETERIZED\_URL

**Example:**

	should_discard 0001 http://${targetSite}/index.html 
	
	Note: the $targetSite variable is replaced by the value passed to test.sh with the -s switch

---

`should_redirect`

  + Use this method when you need to validate that a given request should be redirected to a specific destination

**Usage:**

  should\_redirect TEST\_CASE\_NUMBER SOURCE\_PARAMETERIZED\_URL DEST\_PARAMETERIZED\_URL

**Example:**

	should_redirect 0001 http://${targetSite}/original/location http://${targetSite}/new/location

---

`should_not_redirect`

  + Use this method when you need to validate that a given request is not redirected

**Usage:**

  should\_not\_redirect TEST\_CASE\_NUMBER SOURCE\_PARAMETERIZED\_URL 

**Example:**

	should_not_redirect 0001 http://${targetSite}/original/location

---

`should_rewrite`

  + Use this method when you need to validate that a given request URI is rewritten before being passed to pool member

**Usage:**

  should\_rewrite TEST\_CASE\_NUMBER SOURCE\_PARAMETERIZED\_URL TARGET\_URI

**Example:**

	should_rewrite 0001 http://${targetSite}/original/location /new/location

---

`should_redirect_host`

  + Use this method when you need to validate that a request with a given host header should be redirected to a specific destination

**Usage:**

  should\_redirect\_host TEST\_CASE\_NUMBER SOURCE\_PARAMETERIZED\_URL HOST\_HEADER DEST\_PARAMETERIZED\_URL

**Example:**

        should_redirect_host 0001 http://${targetSite}/original/location shop.acme.com http://${targetSite}/new/location

---

`should_select_pool`

  + Use this method when you need to validate that a given request is sent to a specific pool

**Usage:**

  should\_select\_pool TEST\_CASE\_NUMBER PARAMETERIZED\_URL DEST\_POOL\_NAME

**Examples:**

	should_select_pool 0001 http://${targetSite}/original/location acme_prd_pool

Note: You can also define a variable in the variables section at the top of test.sh that contains the valid pool name.  This makes the test case portable between different iRule environments.

	test $targetSite == "www.acme.com"     && acmePool=acme-prd-pool
	test $targetSite == "web-qa.acme.com"  && acmePool=acme-qa-pool
	test $targetSite == "web-dev.acme.com" && acmePool=acme-dev-pool

	should_select_pool 0001 http://${targetSite}/original/location $acmePool

---

`should_serve_content`

  + Use this method when you need to validate that a given request is responded to directly by the F5.  A common use case for this is serving sorry content from the F5.

**Usage:**

  should\_serve\_content TEST\_CASE\_NUMBER PARAMETERIZED\_URL

**Examples:**

        should_serve_content 0001 http://${targetSite}/sorrypage/index.html


### - http-response.irule - 

This iRule can be used to insert a cookie into all HTTP responses from pool 
selections.  This enables irule-tester to identify which pool was used to 
serve the content for a requested URL

	when HTTP_REQUEST {
		set holder [HTTP::uri]
	}

	when HTTP_RESPONSE {
	
		# Insert LastSelectedPool cookie with the value of the selected pool to
		# enable automated testing to identify if the LB made the expected decision
		HTTP::cookie insert name "LastSelectedPool" value [LB::server pool]
	
		# Insert LastURI cookie with the value of [HTTP::uri] to
		# enable automated testing to identify if the LB made the expected rewrite.
		HTTP::cookie insert name "LastURI" value $holder
	}

## Author

Author: Jesse Mauntel (maunteljw@gmail.com)
