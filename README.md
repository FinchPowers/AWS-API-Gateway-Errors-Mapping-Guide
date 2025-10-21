# AWS API Gateway Errors Mapping Guide

This guide aims at mapping API Gateway errors to their root causes. It takes the approach of an HTTP call, document it,
document the error response and document the possible root causes. A good way to use this guide is to search for errors
directly.

The HTTP calls are made with the [httpie](https://httpie.io/docs/cli) cli. The Returned status code as well as the
`x-amzn-ErrorType` are rendered as part of the error message as they are helpful in identifying the issues.

This guide may not work if API Gateway Gateway Responses are non default.

## Missing Authorizer Identity sources

```shell
http https://<demo-url>/things

HTTP/1.1 401 Unauthorized
x-amzn-ErrorType: UnauthorizedException

{
    "message": "Unauthorized"
}
```

### Possible Cause

Given an authorizer configured to be based on a given `Identity sources`, say the header `Authorization`, and
submitting a request without that identity source.

#### Fix

Send in the header. Eg.: `http https://<demo-url>/things "Authorization: <value>"`

## Invalid or Disabled API Key | No or Improper Usage Plan

```shell
http https://<demo-url>/things "Authorization: <key>"

HTTP/1.1 403 Forbidden
x-amzn-ErrorType: ForbiddenException

{
    "message": "Forbidden"
}
```

### Possible Cause 1 - Non Existing or Disabled API Key

Given an authorizer that sets the api key directly from the authorization header, a requests made with a non existing
key or a disabled key.

#### Fix

Create the API Key in API Gateway, map it to a usage plan tied to the API linked to the request. If it already exists
but is disabled, enable it.

### Possible Cause 2 - Incorrect Authorizer Function

The authorizer lambda function does not set the `usageIdentifierKey` value properly in its reponse.

#### Fix

Fix the authorizer lambda code to extract the right api key value and set it as `usageIdentifierKey` in the response.

### Possible Cause 3 - Invalid Usage Plan

The api key exists and is enabled, but is either not tied to any usage plan, or is tied to a usage plan not including
the target API Gateway.

#### Fix

Associate the API Key with a usage plan that includes the targeted API Gateway.

### Possible Cause 4 - API Gateway API key source miss configured

If you're using an authorizer lambda, API Gateway API key source default value, Header, will not work.

#### Fix

The "Api key source" configuration must be set to "Authorizer".

## Access Denied Explicitly

```shell
http https://<demo-url>/things "Authorization: <key>"

HTTP/1.1 403 Forbidden
x-amzn-ErrorType: AccessDeniedException
...
{
    "message": "User is not authorized to access this resource with an explicit deny"
}
```

### Possible Cause

The lambda authorizer response `policyDocument` contains something like:

```shell
{
    "Version": "2012-10-17",
    "Statement": [{"Action": "execute-api:Invoke", "Effect": "Deny", "Resource": "*"}]
}
```

#### Fix

Check the authorizer code, should it deny or allow the request?

## Access Denied Implicitly

```shell
http https://<demo-url>/things "Authorization: <key>"

HTTP/1.1 403 Forbidden
x-amzn-ErrorType: AccessDeniedException
...
{
    "message": "User is not authorized to access this resource"
}
```

### Possible Cause

The lambda authorizer response `policyDocument` does not contain a resource that matches the one requested. As an
example, if the authorizer response was

```shell
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": "execute-api:Invoke",
            "Effect": "Allow",
            "Resource": [
                "arn:aws:execute-api:*:*:test/test/GET/stuff-is-not-things*"
            ]
        }
    ]
}
```

There is nothing that explicitly matches against `GET /things`, so it's an implicit Access Denied.

#### Fix

Check the authorizer code, should it allow the request?

## Unexisting Method + Route - Scenario 1: Authorization header

```shell
http https://<demo-url>/does-not-exist "Authorization: Bearer <token>"

HTTP/1.1 403 Forbidden
x-amzn-ErrorType: IncompleteSignatureException

{
    "message": "<token> not a valid key=value pair (missing equal-sign) in Authorization header: 'Bearer <token>'"
}
```

### Possible Cause of Unexisting Method + Route - All Scenarios

Calling a Method + Route combination that is undefined in API Gateway. Using httpie, it's especially easy to call a
route with the POST method instead of the GET method by using one instead of two equality signs in the request. Eg.:
`http https://<demo-url>/does-not-exist "Authorization: Bearer <token>" param=value` is a POST call because there is a
single equality sign between `param` and `value`.
`http https://<demo-url>/does-not-exist "Authorization: Bearer <token>" param==value` is a GET call because there are
two equality signs between `param` and `value`.

#### Fix

* Adjut the HTTP call if it was not targetting the right method and route.
* Add the route to API Gateway if it's mising.
* API Gateway yields a 403 out of common security practices. (Basically, obfuscate the unexisting route behing a
  Forbidden error instead of telling an attacker to move on and try to find an existing route.) If it annoys you, you
  can create a catch all route (`{proxy+}`) and make it yield a 404 status all the time. See: [Set up a proxy
  integration with a proxy resource](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-set-up-simple-proxy.html)

## Unexisting Method + Route - Scenario 2: Authorization not via Header

```shell
http https://<demo-url>/does-not-exist "key==<key>"

HTTP/1.1 403 Forbidden
x-amzn-ErrorType: MissingAuthenticationTokenException

{
    "message": "Missing or invalid token"
}
```

### Possible Cause And Fix

See [Possible Cause of Unexisting Method + Route - All Scenarios](
#possible-cause-of-unexisting-method--route---all-scenarios).

## Unexisting Method + Route - Scenario 3: No Authorization Information

```shell
http https://<demo-url>/does-not-exist

HTTP/1.1 400 Bad Request

{
    "message": "Missing or invalid token"
}
```

### Possible Cause And Fix

See [Possible Cause of Unexisting Method + Route - All Scenarios](
#possible-cause-of-unexisting-method--route---all-scenarios).


## GET Request With a Body

```shell
http GET https://<demo-url>/things data=value

HTTP/1.1 403 Forbidden

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<HTML><HEAD><META HTTP-EQUIV="Content-Type" CONTENT="text/html; charset=iso-8859-1">
<TITLE>ERROR: The request could not be satisfied</TITLE>
</HEAD><BODY>
<H1>403 ERROR</H1>
<H2>The request could not be satisfied.</H2>
<HR noshade size="1px">
Bad request.
We can't connect to the server for this app or website at this time. There might be too much traffic or a configuration error. Try again later, or contact the app or website owner.
<BR clear="all">
If you provide content to customers through CloudFront, you can find steps to troubleshoot and help prevent this error by reviewing the CloudFront documentation.
<BR clear="all">
<HR noshade size="1px">
<PRE>
Generated by cloudfront (CloudFront)
Request ID: <some id>
</PRE>
<ADDRESS>
</ADDRESS>
</BODY></HTML>
```

### Possible Cause

Sending a GET request with a body is not supported by AWS.

#### Fix

Reconfigure your endpoint and function code to handle POST instead of GET.
