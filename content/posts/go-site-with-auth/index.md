+++
date = '2025-06-06T09:29:17-06:00'
draft = false
title = 'Making an Go site with OAuth & Sessions'
categories = ["Golang"]
+++

This series covers how you would make a protected web application in Go. Specifically, the website we'll make will use [Oauth 2.0 with PKCE](https://blog.postman.com/what-is-pkce/) to let users log in to our app. Note that I won't cover how to add TLS (i.e. allowing HTTPS requests directly to the server) since that's more easily accomplished with a reverse router like [Caddy](https://caddyserver.com/).

## What is OAuth? Why do we need it?

There are things on the internet not everyone should have access to: your Google Drive, personal employee data, and a ton more. However, we sometimes want to let 3rd party applications access them on our behalf. For example, maybe an app needs access to your Google Drive to store files there, or a company's team needs to use data from another team's microservice.

How can we let those 3rd party apps access sensitive data? We could just give them our Google/Company username and password but that's obviously prone to misuse. Instead, what we can do is let the user log in to some trusted authentication service, get some sort of temporary access token and give it to the 3rd party app. If the 3rd party app was a web application, that might look like:

1. From the 3rd party website, we redirect the user to the authentication service, adding some query parameters to know what 3rd party app is asking for it and where to redirect the user back. The 3rd party app pre-regsiters itself with the authentication server so it knows it's legit.
2. The user logs in to the authentication service on the authentication server's website.
3. The auth server redirects the user back to the 3rd party website, with an access token query parameter to let the app access the sensitive data.

This is called the [implicit grant](https://developer.okta.com/blog/2018/05/24/what-is-the-oauth2-implicit-grant-type). While it works, it has some security flaws. The main issue is the access token is stored in the browser history (it was in the URL, after all) so other users on that computer could steal it by just hitting the back button. If the auth server puts a time limit on the access token, the user must then re-login as there's no way to "refresh" the token.

Some smart people got together and made a newer better flow called OAuth 2.0 with PKCE. While there are other flows, this one can be used with both Single Page Applications (so the user's browser can handle pretty much all of it) or in normal client-server apps. There's a [great readup by Okta](https://developer.okta.com/blog/2019/08/22/okta-authjs-pkce) on how this works, but in summary:

1. When the user asks the app to login to the auth server, the app generates a random string called a code_verifier and a hash of that called a code_challenge. 
2. The app redirects the user to the auth server with the client id, redirect uri, code_challenge, and some other variables for security. It can also have a scope parameter to ask for specific data.
3. The user logs into the auth service on the auth server's website. The auth server also stores the code_challenge for a future step.
4. The auth server redirects the user back to the 3rd party website, with an access code as a query parameter.
5. The app then directly asks the auth server for an access token. It passes along the unhashed code_verifier. The auth server then hashes this and compares it to the code_challenge stored previously to verify that this request is in fact coming from the trusted 3rd party app.
6. The auth server returns an access token to the 3rd party app. It optionally returns a refresh token to ask for more access tokens in the future. 
7. The 3rd party app uses the access tokens to request the secure data.

While more complicated than an implicit flow, OAuth + PKCE is much more secure and is what we'll be using.

## What is a web server?

At its most basic, a web server is something that listens to a [port](https://superuser.com/questions/383763/the-physical-aspect-of-networking-ports) for incoming requests and responds to the requester. 

Sometimes our web server has multiple users talking to it at once. We want to be able to keep track of who's who, otherwise we might send the wrong user the wrong data. Thankfully, browsers have something called "cookies" which they can send with requests. Our web server should be able to look through the cookies send with the request for a special cookie we can call "session" for a unique id. We can then use that id as a key in a lookup table for information on the user in question. If the request has no session cookie we should also give the user one in our response.

Once we can distingish one user from another, we want our web server to send the right web pages depending on what the user requests. If the user hits the /login page, we want to start off the whole OAuth PKCE process. If the user hits an authenticated page, we want to be able to check if the user is logged in and their identity. Essentially:

1. The user sends a request to our server with a session cookie.
2. Our server first looks up the session cookie and retrieves information related to the user
3. The server calls the right function for the requested page, passing along the information from the session
4. We return the response, with a "set-cookie" header as needed.

## Writing a basic server

Let's start with the most basic server we can in Go. Go has a standard library package for web servers that works quite well: net/http. The two main functions we'll use are `http.ListenAndServe()` and `http.NewServeMux()`. 

`ListenAndServe()` takes two arguments: a port to listen on, and an struct with a `ServeHTTP(ResponseWriter, Request)` function that it calls immediately when a new request comes in. 

A `ServeMux` is a convenience object that calls further functions based on a map between route names and structs that also have `ServerHTTP` functions. You add entries to the map via `myServeMux.Handle("/someRoute", myStructWithServeHTTP)`. If it's just a single function you want to pass it to that doesn't require an entirely new struct, you can use `myServeMux.HandleFunc("/someRoute", myFunc)`. A very basic server might look like this:

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    servemux := http.NewServeMux()

    servemux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
    })

    http.ListenAndServe(":8080", servemux)

}
```

Let's start by adding session management. Remember that `ListenAndServe` takes any struct with a `ServeHTTP` function. Let's make our own struct that stores all the information our server might need to handle the request. We'll start small with what it might need:

```go
type Flow struct {
    Session       *scs.SessionManager
    *http.ServeMux
}
```

We'll use the [scs](https://github.com/alexedwards/scs) library to handle session management for us. To make sure every request gets its session handled, we'll want to use the `scs.SessionManager.LoadAndSave(app)` function. This does a few things:

1. Set the "Vary" header in the response to "Cookie". This is mostly just good practice for caching, letting the browser know that the content of the response varies depending on the cookies sent.
2. Retrieve the cookie called "token" from the request, if exists.
3. If the request context (explained below) already has a session value for some reason, use the request as-is for later steps.
4. If no "token" cookie was found, add a new session data object to the context. If there was, but looking it up in the "store"(also explained below) didn't find anything, add a new session data object to the context. If data was found in the store, add that data to the context.
5. Call the `ServeHTTP` function of the passed app. 
6. Save a modified context to the store.
7. Put the set-cookie header in the response with the token if modified, or an empty string if the context was set to destroyed.

Context is essentially the metadata passed along with the request. While the request's body and type are the most used part of the Request object, we can store other parts too. SessionManager adds a unique key to the Request object to stuff the session data in.

Here's a small complete example:
```go
package main

import (
    "fmt"
    "net/http"

    "github.com/alexedwards/scs/v2"
)

type Flow struct {
    Session  *scs.SessionManager
    ServeMux *http.ServeMux
}

func main() {
    app := &Flow{
        ServeMux: http.NewServeMux(),
        Session:  scs.New(),
    }

    app.Session.Cookie.Persist = false // Only lasts as long as the browser is open

    app.ServeMux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        sessionVar := app.Session.GetString(r.Context(), "sessionVar")
        if sessionVar == "" {
            app.Session.Put(r.Context(), "sessionVar", r.URL.Path)
            fmt.Fprintln(w, "No session variable found! Putting it now, refresh the page.")
            return
        } else {
           fmt.Fprintf(w, "Session variable: %s\n", sessionVar)
        }
    })

    fmt.Println("Starting server on :8080")
    http.ListenAndServe(":8080", app.Session.LoadAndSave(app.ServeMux))
}
```

Play around with this in your browser: refreshing the page, setting Cookies.Persist back to true, deleting the site's cookies, and see what happens.

As it is, SessionManager stores all the user sessions informations as part of the object. For scalability, you can set the store to be something like Redis via `mySessionManager.Store = redisstore.New(&redis.Pool{...})` Read the repo's github page for more information.

## Adding authentication

Now we can actually protect some web pages! When the user goes to our `/login` endpoint, we want to redirect them to the OAuth authentication server, regardless of if they're logged in. Go's `golang.org/x/oauth2` package has a `Config` struct that handles a lot of the oauth URL stuff. To begin, let's make an instance of a Config:
```go
import (
...
"golang.org/x/oauth2"
...
)
...
ourApp.Config = &oauth2.Config{
			ClientID:     os.Getenv("CLIENT_ID"),
			ClientSecret: os.Getenv("CLIENT_SECRET"),
			RedirectURL:  flow.RedirectURI.String(),
			Scopes:       scopes,
			Endpoint:     flow.Endpoint.OAuth,
		}
```

There's a Go utility tool called `godotenv` that you install with `go install godotenv`. By running `godotenv go run ./yourFolder`, it automatically pulls environment variables from a `.env` file in the current directory.

Now let's write the login endpoint handler. Remember that for OAuth we need to create a `code\_verifier` and `code\_challenge`. We want to store this` code\_verifier` in our session manager so when the user is redirected back to our site we can extract it and use it for requesting the auth token.  

If the user was going directly to a protected page (e.g. `/profile`), when the user gets redirected back from the auth server we'd like to be able to send them back to the profile page. We can do this by adding a `redirect` query parameter and storing it in the session manager.

The OAuth protocol has an additional query parameter, `state`, that we should check for extra safety (although it's debatable this is actually required?). State is a random string we add to the redirect to the auth server to prevent csrf attacks.

With everything in mind, we can now write our login endpoint:

```go
func (f *OurApp) handleSignIn(w http.ResponseWriter, r *http.Request) {
	state := r.URL.Query().Get("state")
	generatedState := false
	if state == "" {
		state, _ = generateSecureString(16)
		generatedState = true
	}
	redirect := r.URL.Query().Get("redirect")
	if redirect == "" {
		redirect = f.DefaultRedirectPath
	}

	// Generate code_verifier
	verifier := oauth2.GenerateVerifier()
	// Store the verifier, state, and redirect URl to the session for later
	// Store it in the map as "oauth_flow"
	f.Session.Put(r.Context(), "oauth_flow", FlowState{
		Verifier:       verifier,
		State:          state,
		GeneratedState: generatedState,
		Redirect:       redirect,
	})

	// use Go's Config to create the URL we redirect the user with. 
	// AccessTypeOffline lets us get a refresh token.
	authCodeURL := f.Config.AuthCodeURL(state,
		oauth2.AccessTypeOffline,
		oauth2.S256ChallengeOption(verifier))

	http.Redirect(w, r, authCodeURL, http.StatusTemporaryRedirect)
}
```
With that, the user can go to `/login`, get redirected to the auth server, log in, and get redirected back to our specified callback URI with an auth code. Let's write the callback endpoint function to actually get the auth token:
```go
// handleSignInCallback handles the OAuth callback after sign in.
func (f *Flow) handleSignInCallback(w http.ResponseWriter, r *http.Request) {
// Get the auth code from the URL query parameter
	code := r.URL.Query().Get("code")
	if code == "" {
		http.Error(w, "missing code in callback", http.StatusBadRequest)
		return
	}

// Get the verifier, state, and redirect stored in the session manager from 
// the /login endpoint in the map as "oauth_flow"
	if !f.Session.Exists(r.Context(), "oauth_flow") {
		http.Error(w, "missing login attempt", http.StatusBadRequest)
		return
	}
// Remove the data to not mess up future login attempts
	oauthFlow := f.Session.Pop(r.Context(), "oauth_flow").(FlowState)

// Validate state to prevent CSRF.
	if r.URL.Query().Get("state") != oauthFlow.State {
		http.Error(w, "state did not match", http.StatusBadRequest)
		return
	}

// Exchange the authorization code for an access token using the Config struct
	token, err := f.Config.Exchange(context.Background(), code, oauth2.VerifierOption(oauthFlow.Verifier))
	if err != nil {
		http.Error(w, fmt.Sprintf("code exchange failed: %s", err.Error()), http.StatusInternalServerError)
		return
	}

// Renew the session token (not the auth token!) to mitigate session fixation attacks.
	if err = f.Session.RenewToken(r.Context()); err != nil {
		http.Error(w, fmt.Sprintf("failed to renew session token: %s", err.Error()), http.StatusInternalServerError)
		return
	}

// Save the auth token in the context to eventually be saved in the Session store
	f.Session.Put(r.Context(), "token", *token)
// Extract the ID token stored as part of the recieved auth token. 
	idToken := token.Extra("id_token")
	if idToken != nil {
// Store the id token for the sign-out flow.
		f.Session.Put(r.Context(), "id_token", idToken)
	}

// Get the path to redirect to from the stored session information
	redirectWithState := oauthFlow.Redirect
	if !oauthFlow.GeneratedState {
		// Include original state if provided by the caller.
		redirectWithState = appendQueryParams(redirectWithState, map[string]string{"state": oauthFlow.State})
	}

// Redirect the user back to where they wanted to be
	http.Redirect(w, r, redirectWithState, http.StatusSeeOther)
}
```

We extract the auth code from the `code` query parameter. We then check if the session manager can extract the data stored from the user hitting `/login`. We then remove that data from the store to allow future login attempts to work, then validate the state against the state stored in the session manager. We can now exchange the code for a token by asking the auth server with our verifier and code. 

Once that's done, we refresh the session token to prevent [session fixation attacks](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Session_Management_Cheat_Sheet.md#renew-the-session-id-after-any-privilege-level-change), then save the token and id_token in our context (which will be saved into the session manager at the end of the request handling). Finally, we redirect the user to the page they were looking for!

On the id_token: the actual token recieved from the auth code exchange can include additional information about the user depending on what OAuth provider you use. This could be something like the user's name, which is how sites that use a "Log in with..." can get your name (maybe, I haven't double checked).

With the login and login callback pages written up, let's write an actual page. Here's what one might look like:

```go
handleMain := func(w http.ResponseWriter, r *http.Request) {
// Check if the context has an auth token (given to it by the session manager). If not, give a non-logged-in page.
		if app.Session.Exists(r.Context(), "token") {
			// Retrieve the BYU User Info from the ID token
			byuUserInfo, err := app.GetByuUserInfoFromIdToken(r)
			if err != nil {
				http.Error(w, fmt.Sprintf("Failed to get BYU user info: %v", err), http.StatusInternalServerError)
				return
			}

			fmt.Fprintf(w, `<html><body>
			<h2>Welcome Back %s!</h2>
			<a href="/protected/echo">Echo</a>
			<a href="/protected/revoke">Revoke</a>
			<a href="%s">Sign Out</a>
			</body></html>`, byuUserInfo.DisplayName, "/signout")
			return
		}

		// If no token is found, prompt the user to sign in.
		fmt.Fprintf(w, `<html><body>
		<h2>Welcome!</h2>
		<a href="%s">Sign In</a>
		</body></html>`, "/signin")
}

func (f *App) GetByuUserInfoFromIdToken(r *http.Request) (*ByuUserInfo, error) {
	if f.Session.Exists(r.Context(), "id_token_byu_info") {
		// The userInfo was previously decoded and cached
		userInfo := f.Session.Get(r.Context(), "id_token_byu_info").(ByuUserInfo)
		return &userInfo, nil
	}

	var userInfo ByuUserInfo
	err := GetUserInfoFromIdToken(f, r, &userInfo)
	if err != nil {
		return nil, err
	}

	f.Session.Put(r.Context(), sessionKeyBYUUserInfo, userInfo)

	return &userInfo, err
}

func GetUserInfoFromIdToken[T any](f *Flow, r *http.Request, userInfo *T) error {
	if !f.Session.Exists(r.Context(), "id_token") {
		return errors.New("no id_token in the session")
	}

	idToken := f.Session.Get(r.Context(), "id_token").(string)
	if idToken == "" {
		return nil
	}
	parts := strings.Split(idToken, ".")
	if len(parts) != 3 {
		return errors.New("not a jwt token, expected three parts")
	}
	payload, err := base64.RawURLEncoding.DecodeString(parts[1])
	if err != nil {
		return err
	}
	return json.Unmarshal(payload, userInfo)
}
```

Obviously this is just a proof of concept: It's vulnerable to XSS attacks since it directly uses the result from the token in the page. A much better approach would be to use Go's HTML templating engine, but that's outside the scope of this post.

We can now write endpoints that actually use the data we worked so hard to get access to! Inside a handler function, you might want to write something like this:

```go
		// Get the auth token from the request context
		token := f.Session.Get(r.Context(), "token").(oauth2.Token)
		// Make a TokenSource that gives a valid token and, if it's expired, requests a new one.
	baseSource := f.Config.TokenSource(ctx, &token)
// Wrap TokenSource in a ReuseTokenSource for caching reasons, and because TokenSource requesting a new token isn't concurrent-safe. This makes things be ok.
	sessionSource := oauth2.ReuseTokenSource(nil, &scsTokenSource{
		base:    baseSource,
		session: f.Session,
		context: r.Context(),
	})
// Make a new client that can make external requests using the token for authentication.
	client := oauth2.NewClient(r.Context(), sessionSource)
// make the request and save the response
		resp, err := client.Get("https://api.byu.edu/echo/v2/echo")
		if err != nil {
			http.Error(w, fmt.Sprintf("Failed to call echo: %v", err), http.StatusInternalServerError)
			return
		}
		defer resp.Body.Close()

		// Set the Content-Type header to JSON
		w.Header().Set("Content-Type", "application/json")

		// Read the response body and write it to the client.
		bodyBytes, err := io.ReadAll(resp.Body)
		w.Write(bodyBytes)
```

With pretty much everything in the app all set, we can now write our logout endpoint. To actually logout our user, we need to both destroy the session data on our side as well as let the auth server know that we're logging out our user and that the tokens should not work anymore. Based on this, we know we need two endpoints: the actual logout, and the logout callback the auth server calls once its done. The logout endpoint should:

1. Verify we have a session for a given context, throwing an error otherwise
2. Get or generate a state for the CSRF protection
3. Get the redirect from a query parameter or use a defualt
4. Put the state and redirect into the session manager store
5. Redirect the user to the auth server, giving the client id, state, the logut callback uri, and the id token so it knows which session to delete.

Then the callback endpoint should:

1. Verify the session manager has the oauth_token data stored from the logout endpoint on hand
2. Verify the state matches
3. If there was a user-generated state, add it to the redirect URL from the session manager storage
4. Redirect the user

Both fairly straightforward at this point. Here's what it might look like:

```go
// handleSignOut initiates the sign out process.
func (f *Flow) handleSignOut(w http.ResponseWriter, r *http.Request) {
	if !f.Session.Exists(r.Context(), sessionKeyToken) {
		params := map[string]string{
			"error": "missing token",
		}
		http.Redirect(w, r, appendQueryParams(f.ErrorRedirectURI.String(), params), http.StatusSeeOther)
		return
	}

	state := r.URL.Query().Get("state")
	generatedState := false
	if state == "" {
		state, _ = generateSecureString(16)
		generatedState = true
	}
	redirect := r.URL.Query().Get("redirect")
	if redirect == "" {
		redirect = f.DefaultRedirectPath
	}

	if f.Endpoint.EndSessionURL == "" {
		// Cannot end session upstream.
		log.Printf("Cannot end session upstream! This can lead to compromised user sessions!")
		f.Session.Destroy(r.Context())
		http.Redirect(w, r, redirect, http.StatusSeeOther)
		return
	}

	var idToken string
	if !f.Session.Exists(r.Context(), sessionKeyIDToken) {
		f.Session.Destroy(r.Context())
	} else {
		idToken = f.Session.Get(r.Context(), sessionKeyIDToken).(string)
		f.Session.Put(r.Context(), sessionKeyOAuthFlow, FlowState{
			State:          state,
			GeneratedState: generatedState,
			Redirect:       redirect,
		})
	}

	params := map[string]string{
		"id_token_hint":            idToken,
		"post_logout_redirect_uri": f.PostLogoutRedirectURI.String(),
		"client_id":                f.Config.ClientID,
		"state":                    state,
	}
	endSessionUrl := appendQueryParams(f.Endpoint.EndSessionURL, params)
	http.Redirect(w, r, endSessionUrl, http.StatusSeeOther)
}

// handleSignOutCallback handles the OAuth callback after sign out.
func (f *Flow) handleSignOutCallback(w http.ResponseWriter, r *http.Request) {
	if !f.Session.Exists(r.Context(), sessionKeyOAuthFlow) {
		http.Error(w, "missing logout attempt", http.StatusBadRequest)
		return
	}
	oauthFlow := f.Session.Pop(r.Context(), sessionKeyOAuthFlow).(FlowState)

	if r.URL.Query().Get("state") != oauthFlow.State {
		http.Error(w, "state did not match", http.StatusBadRequest)
		return
	}

	redirectWithState := oauthFlow.Redirect
	if !oauthFlow.GeneratedState {
		redirectWithState = appendQueryParams(redirectWithState, map[string]string{"state": oauthFlow.State})
	}
	f.Session.Destroy(r.Context())
	http.Redirect(w, r, redirectWithState, http.StatusSeeOther)
}
```

And we're done! You now have a site with a login page, protected pages, and a logout page. The next step would be to organize the handler functions and use Go's templating engine. If you have other servers you own that you want to communicate with you could add the [client credentials](https://auth0.com/docs/get-started/authentication-and-authorization-flow/client-credentials-flow) OAuth flow. The world is your oyster!