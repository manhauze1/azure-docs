---
title: include file
description: include file
services: active-directory
documentationcenter: dev-center-name
author: navyasric
manager: CelesteDG
editor: ''

ms.service: active-directory
ms.devlang: na
ms.topic: include
ms.tgt_pltfrm: na
ms.workload: identity
ms.date: 09/17/2018
ms.author: nacanuma
ms.custom: include file
---

## Use the Microsoft Authentication Library (MSAL) to sign in the user

1. Add the following code to your `index.html` file within the `<script></script>` tags:

```javascript
//Pass null for default authority (https://login.microsoftonline.com/common)
var myMSALObj = new Msal.UserAgentApplication(applicationConfig.clientID, null, acquireTokenRedirectCallBack,
    {storeAuthStateInCookie: true, cacheLocation: "localStorage"});

function signIn() {
    myMSALObj.loginPopup(applicationConfig.graphScopes).then(function (idToken) {
        //Login Success
        showWelcomeMessage();
        acquireTokenPopupAndCallMSGraph();
    }, function (error) {
        console.log(error);
    });
}

function acquireTokenPopupAndCallMSGraph() {
    //Call acquireTokenSilent (iframe) to obtain a token for Microsoft Graph
    myMSALObj.acquireTokenSilent(applicationConfig.graphScopes).then(function (accessToken) {
        callMSGraph(applicationConfig.graphEndpoint, accessToken, graphAPICallback);
    }, function (error) {
        console.log(error);
        // Call acquireTokenPopup (popup window) in case of acquireTokenSilent failure due to consent or interaction required ONLY
        if (error.indexOf("consent_required") !== -1 || error.indexOf("interaction_required") !== -1 || error.indexOf("login_required") !== -1) {
            myMSALObj.acquireTokenPopup(applicationConfig.graphScopes).then(function (accessToken) {
                callMSGraph(applicationConfig.graphEndpoint, accessToken, graphAPICallback);
            }, function (error) {
                console.log(error);
            });
        }
    });
}

function graphAPICallback(data) {
    //Display user data on DOM
    var divWelcome = document.getElementById('WelcomeMessage');
    divWelcome.innerHTML += " to Microsoft Graph API!!";
    document.getElementById("json").innerHTML = JSON.stringify(data, null, 2);
}

function showWelcomeMessage() {
    var divWelcome = document.getElementById('WelcomeMessage');
    divWelcome.innerHTML += 'Welcome ' + myMSALObj.getUser().name;
    var loginbutton = document.getElementById('SignIn');
    loginbutton.innerHTML = 'Sign Out';
    loginbutton.setAttribute('onclick', 'signOut();');
}

// This function can be removed if you do not need to support IE
function acquireTokenRedirectAndCallMSGraph() {
    //Call acquireTokenSilent (iframe) to obtain a token for Microsoft Graph
    myMSALObj.acquireTokenSilent(applicationConfig.graphScopes).then(function (accessToken) {
      callMSGraph(applicationConfig.graphEndpoint, accessToken, graphAPICallback);
    }, function (error) {
        console.log(error);
        //Call acquireTokenRedirect in case of acquireToken Failure
        if (error.indexOf("consent_required") !== -1 || error.indexOf("interaction_required") !== -1 || error.indexOf("login_required") !== -1) {
            myMSALObj.acquireTokenRedirect(applicationConfig.graphScopes);
        }
    });
}

function acquireTokenRedirectCallBack(errorDesc, token, error, tokenType) {
    if (tokenType === "access_token") {
        callMSGraph(applicationConfig.graphEndpoint, token, graphAPICallback);
    } else {
        console.log("token type is:"+tokenType);
    }
}


// Browser check variables
var ua = window.navigator.userAgent;
var msie = ua.indexOf('MSIE ');
var msie11 = ua.indexOf('Trident/');
var msedge = ua.indexOf('Edge/');
var isIE = msie > 0 || msie11 > 0;
var isEdge = msedge > 0;

//If you support IE, our recommendation is that you sign-in using Redirect APIs
//If you as a developer are testing using Microsoft Edge InPrivate mode, please add "isEdge" to the if check
if (!isIE) {
    if (myMSALObj.getUser()) {// avoid duplicate code execution on page load in case of iframe and popup window.
        showWelcomeMessage();
        acquireTokenPopupAndCallMSGraph();
    }
} else {
    document.getElementById("SignIn").onclick = function () {
        myMSALObj.loginRedirect(applicationConfig.graphScopes);
    };
    if (myMSALObj.getUser() && !myMSALObj.isCallback(window.location.hash)) {// avoid duplicate code execution on page load in case of iframe and popup window.
        showWelcomeMessage();
        acquireTokenRedirectAndCallMSGraph();
    }
}
```

<!--start-collapse-->
### More Information

After a user clicks the **Sign In** button for the first time, the `signIn` method calls `loginPopup` to sign in the user. This method results in opening a popup window with the *Microsoft Azure Active Directory v2.0 endpoint* to prompt and validate the user's credentials. As a result of a successful sign-in, the user is redirected back to the original *index.html* page, and a token is received, processed by `msal.js` and the information contained in the token is cached. This token is known as the *ID token* and contains basic information about the user, such as the user display name. If you plan to use any data provided by this token for any purposes, you need to make sure this token is validated by your backend server to guarantee that the token was issued to a valid user for your application.

The SPA generated by this guide calls `acquireTokenSilent` and/or `acquireTokenPopup` to acquire an *access token* used to query the Microsoft Graph API for user profile info. If you need a sample that validates the ID token, take a look at [this](https://github.com/Azure-Samples/active-directory-javascript-singlepageapp-dotnet-webapi-v2 "GitHub active-directory-javascript-singlepageapp-dotnet-webapi-v2 sample") sample application in GitHub – the sample uses an ASP.NET Web API for token validation.

#### Getting a user token interactively

After the initial sign-in, you do not want to ask users to reauthenticate every time they need to request a token to access a resource – so *acquireTokenSilent* should be used most of the time to acquire tokens. There are situations however that you need to force users to interact with Azure Active Directory v2.0 endpoint – some examples include:

- Users may need to reenter their credentials because the password has expired
- Your application is requesting access to a resource that the user needs to consent to
- Two factor authentication is required

Calling the *acquireTokenPopup(scope)* results in a popup window (or *acquireTokenRedirect(scope)* results in redirecting users to the Azure Active Directory v2.0 endpoint) where users need to interact by either confirming their credentials, giving the consent to the required resource, or completing the two factor authentication.

#### Getting a user token silently

The ` acquireTokenSilent` method handles token acquisitions and renewal without any user interaction. After `loginPopup` (or `loginRedirect`) is executed for the first time, `acquireTokenSilent` is the method commonly used to obtain tokens used to access protected resources for subsequent calls - as calls to request or renew tokens are made silently.
`acquireTokenSilent` may fail in some cases – for example, the user's password has expired. Your application can handle this exception in two ways:

1. Make a call to `acquireTokenPopup` immediately, which results in prompting the user to sign in. This pattern is commonly used in online applications where there is no unauthenticated content in the application available to the user. The sample generated by this guided setup uses this pattern.

2. Applications can also make a visual indication to the user that an interactive sign-in is required, so the user can select the right time to sign in, or the application can retry `acquireTokenSilent` at a later time. This is commonly used when the user can use other functionality of the application without being disrupted - for example, there is unauthenticated content available in the application. In this case, the user can decide when they want to sign in to access the protected resource, or to refresh the outdated information.

> [!NOTE]
> The above code uses the `loginRedirect` and `acquireTokenRedirect` methods when the browser used is Internet Explorer due to a [known issue](https://github.com/AzureAD/microsoft-authentication-library-for-js/wiki/Known-issues-on-IE-and-Edge-Browser) related to handling of popup windows by Internet Explorer browser.
<!--end-collapse-->

## Call the Microsoft Graph API using the token you just obtained

Add the following code to your `index.html` file within the `<script></script>` tags:

```javascript
function callMSGraph(theUrl, accessToken, callback) {
    var xmlHttp = new XMLHttpRequest();
    xmlHttp.onreadystatechange = function () {
        if (this.readyState == 4 && this.status == 200)
            callback(JSON.parse(this.responseText));
    }
    xmlHttp.open("GET", theUrl, true); // true for asynchronous
    xmlHttp.setRequestHeader('Authorization', 'Bearer ' + accessToken);
    xmlHttp.send();
}
```
<!--start-collapse-->

### More information on making a REST call against a protected API

In the sample application created by this guide, the `callMSGraph()` method is used to make an HTTP `GET` request against a protected resource that requires a token and then return the content to the caller. This method adds the acquired token in the *HTTP Authorization header*. For the sample application created by this guide, the resource is the Microsoft Graph API *me* endpoint – which displays the user's profile information.

<!--end-collapse-->

## Add a method to sign out the user

Add the following code to your `index.html` file within the `<script></script>` tags:

```javascript
/**
 * Sign out the user
 */
 function signOut() {
     myMSALObj.logout();
 }
```
