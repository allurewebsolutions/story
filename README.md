# story

I feel like every day is an important learning experience in my life as a developer. But I will tell you one particular story about how I needed to implement a new solution for google login for our Cordova iOS app.

It went smoothly for Android by following the documentation: https://github.com/EddyVerbruggen/cordova-plugin-googleplus

The code looked like this:
```
    googleLogin: function() {
        var data;
        window.openLoader({'text': 'Logging In...'});
        window.plugins.googleplus.login(
            {
                'scopes': 'https://www.googleapis.com/auth/plus.login https://www.googleapis.com/auth/plus.profile.emails.read',
                'webClientId': '617421937193-dbfsdngr0j1i3g10ecdum7ugjf92t6r6.apps.googleusercontent.com',
                'offline': true
            },
            function (obj) {
                window.localStorage.googleUserInfo = JSON.stringify(obj);
                callback = function(status) {
                    if (status) {
                        cordovaApp.ifRegistration(obj.idToken, cordovaApp.getGoogleUserInfo);
                    }
                    else {
                        window.closeLoader();
                        myApp.alert('Something went wrong. Please try again with Google.', 'Error');
                    }
                };
                $.post('https://www.googleapis.com/oauth2/v4/token', {
                    code: obj.serverAuthCode,
                    client_id: '617421937193-dbfsdngr0j1i3g10ecdum7ugjf92t6r6.apps.googleusercontent.com',
                    client_secret: 'ZtGIoJqxu6cljcGNJLS_0WO3',
                    redirect_uri: 'http://localhost',
                    grant_type: 'authorization_code'
                }).done(function(data) {
                    // after that make the login api request to own site
                    makeApiRequest('auth/oauth', {'token': data.access_token, 'provider': 'google'}, 'POST')
                        .success(function(response) {
                            typeof callback === 'function' && callback(true);
                        })
                        .error(function() {
                            typeof callback === 'function' && callback(false);
                        });
                }).fail(function(response) {
                    window.closeLoader();
                    myApp.alert('Something went wrong. Please try again with Google.', 'Error');
                });
            },
            function (msg) {
                window.closeLoader();
                myApp.alert('Something went wrong. Please try again with Google.', 'Error');
            }
        );

    },
 ```
 
Everything should have worked for iOS too, but it didn't. And the documentation didn't say anything useful, nor did Google searches. I was stuck and needed to debug deeply. In order to figure out what was happening, I decided to take a look at the response I was receiving from Google when logging in on iOS. This meant I needed to open the Safari simulator and look at the API response and it turned out at the API response on iOS was different that with Android, but it wasn't clear exactly what parameters it was requiring or accepting.

I took another look at the error I was receiving: `Invalid parameter value for redirect_uri` - This gave me the idea perhaps the parameters we are posting to oauth2 are different for iOS and Android. I decided to bypass the post entirely if the platform was iOS:
 
 ```
 if(device.platform == 'iOS') {
                    makeApiRequest('auth/oauth', {'token': obj.accessToken, 'provider': 'google'}, 'POST')
                        .success(function(response) {
                            typeof callback === 'function' && callback(true);
                        })
                        .error(function() {
                            typeof callback === 'function' && callback(false);
                        });
                }
```
It turns out that for iOS only the reverse_client_id is needed as a parameter which is defined in the Cordova config.xml file.

The final function looked like this:
```
    googleLogin: function() {
        var data;
        window.openLoader({'text': 'Logging In...'});
        window.plugins.googleplus.login(
            {
                'scopes': 'https://www.googleapis.com/auth/plus.login https://www.googleapis.com/auth/plus.profile.emails.read',
                'webClientId': '617421937193-dbfsdngr0j1i3g10ecdum7ugjf92t6r6.apps.googleusercontent.com',
                'offline': true
            },
            function (obj) {
                window.localStorage.googleUserInfo = JSON.stringify(obj);
                callback = function(status) {
                    if (status) {
                        cordovaApp.ifRegistration(obj.idToken, cordovaApp.getGoogleUserInfo);
                    }
                    else {
                        window.closeLoader();
                        myApp.alert('Something went wrong. Please try again with Google.', 'Error');
                    }
                };

                if(device.platform == 'iOS') {
                    makeApiRequest('auth/oauth', {'token': obj.accessToken, 'provider': 'google'}, 'POST')
                        .success(function(response) {
                            typeof callback === 'function' && callback(true);
                        })
                        .error(function() {
                            typeof callback === 'function' && callback(false);
                        });
                }else{
                    $.post('https://www.googleapis.com/oauth2/v4/token', {
                        code: obj.serverAuthCode,
                        client_id: '617421937193-dbfsdngr0j1i3g10ecdum7ugjf92t6r6.apps.googleusercontent.com',
                        client_secret: 'ZtGIoJqxu6cljcGNJLS_0WO3',
                        redirect_uri: 'http://localhost',
                        grant_type: 'authorization_code'
                    }).done(function(data) {
                        // after that make the login api request to own site
                        makeApiRequest('auth/oauth', {'token': data.access_token, 'provider': 'google'}, 'POST')
                            .success(function(response) {
                                typeof callback === 'function' && callback(true);
                            })
                            .error(function() {
                                typeof callback === 'function' && callback(false);
                            });
                    }).fail(function(response) {
                        window.closeLoader();
                        myApp.alert('Something went wrong. Please try again with Google.', 'Error');
                    });
                }

            },
            function (msg) {
                window.closeLoader();
                myApp.alert('Something went wrong. Please try again with Google.', 'Error');
            }
        );

    },
```

## Lesson
Documentation isn't always complete and correct and sometimes you will have to come up with creative ways to solve various technical challenges. In this case, I was lucky that my guess that we didn't need to post any additional paramaters was correct.
