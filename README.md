# FourOhOne

[![Build Status](https://travis-ci.org/Commit451/FourOhOne.svg?branch=master)](https://travis-ci.org/Commit451/FourOhOne) [![](https://jitpack.io/v/Commit451/FourOhOne.svg)](https://jitpack.io/#Commit451/FourOhOne)

`401` errors can be a pain to deal with. Do we retry auth? Do we refresh the user token? Do we show UI for the user to sign in again?

FourOhOne builds off of the OkHttp `Authenticator` class to make it easy to handle 401 errors globally in an app.

## Simple Setup
Setup can be as simple or as complex as your API needs it to be. Here is a pretty straightforward example:
```kotlin
val fourOhOneAuthenticator = FourOhOneAuthenticator.Builder(FourOhOneAuthenticator.Callback() {

    override fun onReauthenticate(route: Route, response: Response): Request? {
        //we got a 401 error. Try refreshing the token
        //don't worry, already on a background thread
        val token = apiClient.refreshCurrentToken()
        //don't forget to set it on the client, so that future calls do not fail
        apiClient.setToken(token);
        //Now, set it on the currently failing request
        Request.Builder requestBuilder = response.request().newBuilder()
        requestBuilder.header("Authorization", token)
        //Try this request with the new header
        return requestBuilder.build()
    }

    override fun onUnableToAuthenticate(Route route, Response response) {
        //We have maxed out our number of retries, we just need to tell the user to reauth
        //We are on a background thread here, so move to the main thread
        postOnMainThread(Runnable {
            Toast.makeText(context, "You have been signed out", Toast.LENGTH_SHORT)
                    .show()

            //relaunch login screen, clearing the stack
            val intent = Intent(context, LoginActivity.class)
            intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK)
            context.startActivity(intent)
        })
    }
})
.build()
val client = new OkHttpClient.Builder()
        .authenticator(fourOhOneAuthenticator)
        .build()
//pass this OkHttp client to Retrofit, or wherever you are using it
```
If you do not have a way of reauthenticating, and just want to go straight to `onUnableToAuthenticate`, return null:
```kotlin
override fun onReauthenticate(route: Route, response: Response): Request? {
    //onUnableToAuthenticate will be called right away
    return null
}
```

## Ignore Calls
There are times where you want to ignore this globally defined behavior and let a 401 be ignored by the `FourOhOneAuthenticator`. This would include calls such as calls to `login` which would 401 if the user typed in the wrong credentials. In order to ignore these requests, you can attach a header to your request which will tell the authenticator to ignore it:
```java
Request loginRequest = new Request.Builder()
    .url(url)
    .addHeader(FourOhOneAuthenticator.HEADER_IGNORE, "blah")
    .build();
```
The value of the header is ignored, just its presence tells the Authenticator to ignore the Request.

This can be done similarly in Retrofit using header annotations, or possibly via an Interceptor if you do not have control over the defined Interface.

You can also change the header to be something custom to suit your needs:
```
FourOhOneAuthenticator fourOhOneAuthenticator = new FourOhOneAuthenticator.Builder(callback)
    .ignoreHeader("ACustomHeaderKey")
    .build();
```

See the `app` module for a somewhat extensive example.

License
--------

    Copyright 2017 Commit 451

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
