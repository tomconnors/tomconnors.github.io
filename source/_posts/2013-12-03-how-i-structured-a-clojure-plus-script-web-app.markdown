---
layout: post
title: "How I Structured a Clojure(+Script) Web App"
date: 2013-12-03 20:38:44 -0500
comments: true
categories: 
---

I've been working on a full-stack Clojure application for about a month now, and have arrived at an application architecture that seems to be passably decent, so I thought I'd share. A caveat: I use Facebook's [React](http://facebook.github.io/react/index.html) as a rendering engine which greatly simplifies client side code at the cost of using a young and heavy library.

<!-- more -->

#### Contents

* [Organization of Files](#organization-of-files)
* [Separation of Responsibilities](#separation-of-responsibilities)
* [An Example Application](#an-example-application)
* [What the Future Holds](#what-the-future-holds)

<a name="organization-of-files"></a>
#### Organization of Files

The structure of the files in the application is probably about what you'd expect from any web app nowadays, with the added wrinkle of a `shared` source directory. I cross-compile using [cljx](https://github.com/lynaghk/cljx), which has treated me well so far. Running `tree` on the `src` dir produces something like:

* src/
	* client/
		* composur/ (the name of the app and thus the name of the top-level namespace)
			* components/
				* app.cljs
				* home.cljs
			* core.cljs
			* routes.cljs
			* models/
				* app.cljs
		* styles/
	* server/
		* composur/
			* models/
			* routes/
				* auth.clj
			* server.clj
			* system.clj
			* layout.clj
			* web_page.clj
	* shared/
		* composur/
			* shared/

##### Regarding the Client-Only Code

The entrance to the client-only code is the namespace `composur.core`. If a file isn't directly or indirectly required by `core`, none of that file's code will survive advanced compilation.

The `components` directory houses what might also be called "views." This is where I define my [React](http://facebook.github.io/react/index.html) components. If you're unfamiliar with React, I recommend taking some time to try it out. The library has a very small and easy to use api, and usage from clojurescript is simple, thanks to [@piranha](https://github.com/piranha) and [pump](https://github.com/piranha/pump). If you're not interested in using React, I'd love to hear how you handle dynamic views in clojurescript.

The `composur.components.app` namespace defines the root component for the application - the one into which all others will be rendered.

In `composur.routes` I define the client-side routes for the application, using a fork of [secretary](https://github.com/gf3/secretary) I made to allow route functions to be React components rather than plain functions. That way, when routing occurs, the new component to render is handed to the top-level component, which can chose where to render it.

In `composur.models.app` I define a global application state atom, similar to, but not nearly as robust as [pedestal](http://pedestal.io/)'s model of application state. In fact, I considered using pedestal for my application, but the framework is still too young to be easy to be productive with. So, I stole some ideas I almost understood.

##### Regarding Server-Only Code

I structured the server-side code in accordance with [Stuart Sierra's workflow](http://thinkrelevance.com/blog/2013/06/04/clojure-workflow-reloaded), and it's really turned out to be worth the (small amount of) initial effort.

As per that workflow, `composur.system` defines functions to start and stop the system as a whole, `composur.server` provides the same functions for the web server portion of it. Other things that might require starting and stopping include database connections.

Interesting things happen in `composur.web-page`; that's where I define the main [ring](https://github.com/ring-clojure/ring) handler, pulling in specific route handlers from files under the `routes` subdirectory. That's also where I apply the [friend](https://github.com/cemerick/friend) authentication middleware.

`layout.clj` defines the html that's delivered to the client. I don't do any server-side rendering except for the body and a div for mounting the main React component. I used [hiccup](https://github.com/weavejester/hiccup) for html templating, although I might have prefered something closer to vanilla html. [Enlive](https://github.com/cgrand/enlive) is a lib that allows transformations of plain html, but hiccup turned out to be much easier to use.

The `routes` subdirectory has handlers for a mostly RESTful api. Above, I listed an `auth.clj` file, because there's an interesting gotcha there that I haven't solved yet. When users log in (using "/login", shockingly) they receive an authenticated response with data for their current route (such as the home page), because the data for that route may have changed if I'm serving different content based on authentication status. I had hoped to reuse my [compojure](https://github.com/weavejester/compojure) routes for this, but that turned out to be tricky. So in `auth` I'm duplicating some routing logic and using the lower-level [clout](https://github.com/weavejester/clout) library to help match routes like "/" to functions like `home-page-data`. Of course, this duplication is bad news, and I'll be looking for a way to get rid of it.

A final note about the server code - routes are handled differently for standard GET requests and ajax GETs. When a browser first requests a page, say "/", the server derives and returns three things: the application-wide data (currently just the user's info, if there is a user logged in), the route-specific data, and the layout html. Subsequent ajax GETs, to, for instance, "/help", will only return the data for the route being requested. 

##### Regarding Shared Code

The code that's currently cross compiled is primarily utility functions and simple data transformations and validations. As the project matures and separation of responsibilities becomes more strict and refined, I hope to move more and even *most* of the code into the shared directory. Speaking of the separation of responsibilities...

<a name="separation-of-responsibilities"></a>
#### Separation of Responsibilities

To use some common clojure-speak, I'm sure I've complected a few concepts in the division of labor of my code, but here's the current outline.

##### State

Server side code is mostly stateless. I've aimed for pure functions wherever possible, breaking from that rule only when interacting with the database. 

The client, however, has much state. As mentioned earlier, I use a single atom to contain all application state. The same namespace that defines the atom defines functions for requesting updates to it. Those functions are just abstractions on top of core.async channels. There's a number of channels for interacting with the state:

* input-chan: accepts messages that look like 

  [[:path :to :relevant :part :of :app-state] :operation-to-perform :any 'args']

  This is the most common way to request changes to the application state. Any application component may write to this channel using the function `put-input!` and the message (slightly tranformed) will be written to both the transform and effect channels.

* transform-chan: accepts messages that look like: 

  [[:path :to :target] :operation old-app-state parameters-map]

  When messages are read from the transform-chan, if a registered transform function can be found that matches the target and operation, that function will be called with `target operation state-at-target old-application-state additional-params`, and the returned value will be `update-in`'d the app-state at the target.

  In my experience, the target and operation parameters are rarely used, but it's good to have them there because transforms can be registered to match paths and operations using wildcards (:* for a single part of the path, :** for zero or more parts of the path - this, and most of the logic for handling state, was inspired by pedestal.)

* effect-chan: accepts messages that look like: 

  [[:path :to :target] :operation old-app-state parameters-map]

  The effect-chan is used to make calls to the server in order to persist changes made on the client. The effect-chan operates in a manner very similar to the transform-chan, except rather than updating local state, it requests updates to external state.

* update-chan: accepts messages that look like 

  [:target new-value]

  The update-chan is used when data is fetched from the server. Because data comes back from the server already in the format I want, and therefore needs no transformation, I simply write to update-chan with `put-update!` and the supplied location in the application state is updated. 

* output-queue: notice that this one is a queue, not a channel. Other parts of the application (primarily view components) may subscribe to the output-queue for changes at a particular path (or paths). If you've ever `add-watch`'d an atom, this queue allows you to do that with the application state atom, but at a higher level of granularity, because you only get updates for paths you care about.

Perhaps my naming convention of 'chan' vs 'queue' isn't clear enough; I'd be happy to hear other ideas.

##### Business Rules

A few things fall under the umbrella of "Business Rules":

* Data validation
* Data transformations
* User authorization

Functions for each of these live under the `models` directory, in the client, server, and shared parent directories. Most of what is split onto the client and server could and likely should be combined into the shared codebase, making it easy to do things like verify that a new user's password is long enough on the client before sending it, then re-verifying it with the same logic on the server. The same sort of thing could be done with data transformations; transform on client so they see immediate results of interactions, and transform on the server with the same logic to derive the data to persist. 

##### CRUD

CRUD also lives under `models`, but only on the server, because it involves the database. In MVC vernacular, you could say this architecture uses a fat model. CRUD is pretty simple with my setup: a request is picked up by a route defined under the `routes` directory, and that handler calls the validation and authorization functions defined in the appropriate `model` namespace, then, if everything checks out, the route handler calls the CRUD function from the same `model` namespace. Sometimes the line between authorization/validation and CRUD is blurred if I can do it all in a single database query. I'm not sure yet whether that pattern will prove 'anti-'.

##### Rendering

React and pump make rendering easy, joyful even. I simply define components with declarative code that describes what they look like when the application is in some state, and 're-render' everything when any application state changes. That sounds slow, I know, but it's working and I haven't had a speed problem yet. Note that React doesn't actually re-render on every change; it only renders what has actually changed on screen.

The root application component sets up a listener on the application state's output queue for all updates, and re-renders when it receives one, which introduces a probable location to optimize, if speed does turn out to be a problem. The root component could listen to only changes that affect the application layout as a whole, and subcomponents could handle finer-grained update subscriptions.

##### Authentication

I use Friend for authentication. On the server, auth code is pretty minimal. There's routes defined under `routes/auth.clj` to handle signup, login, and logout, and some rules and CRUD defined under `models/users.clj`. Routes unrelated to authentication receive a map representing the current user as a parameter, so they can return varying data based on that user's existence and permissions.

On the client, the map describing the current user is just another piece of application state, but because the presence of a user often has an effect on rendering, the current user is threaded through most of the component heirarchy. This fits better with the functional way of doing things, and with react's way of doing things, than trying to access the user as global state would.

#### Addressing Some Common SPA Concerns

One of my biggest beefs with the todo applications that framework and library designers put together is that they generally don't show how to solve many of the common plumbing concerns of single page applications. When I started my application, I had no intention of doing anything new or creative with plumbing, but I couldn't find simple, well documented examples of the things I wanted my application to do, so I had no choice. Because of all that, some of what you're about to read might be drivel. Indeed, some of everything you just read might be drivel.

##### Initialization

Initialization on the server side is incredibly easy, thanks to Stuart Sierra. I run (reset) at my repl, and my server is running.

Client side initialization is a bit different, but still pretty simple. Any namespace that defines code that must run at initialization time defines an `init` function, and `composur.core`'s `init` function calls all of them. I intend to formalize this a bit, and move my client-side code in the direction of Mr. Sierra's workflow. 

##### Data Bootstrap

To avoid an ajax request right after page load to get the initial data, when the server handles a request, the html it returns contains a script tag with a type of "text/edn", which contains the initial application state, and which is read at initialization time on the client. The event that causes rendering to start on the client is this data being written to the application state via `put-update!`

##### Routing and History Management

It's important to note that routing and history management are distinct but related things. History management concerns dealing with the back and forward buttons, and adding history entries when the user navigates within your page. Routing is responding to changes in the url, generally by rendering a different page.

I use [history.js](https://github.com/browserstate/history.js/) for handling back and forward button clicks, as well as updating the url based on in-app navigation. I'll glady switch to google closure's history library, just as soon as they support non-hash based urls. My app uses urls like "/users/userid", whereas google closure would give me "/users#userid". Such frivolities cost web developers hours of sleep.

When navigation events occur, a series of things happens: 

* the url is updated right away. 
* a listener responds by fetching the data for the new route.
* Once the data comes back, the app state is first updated with the data, then with the new route. The order is important, because changing the route before changing the data would cause, in the best case, an unneeded actual DOM manipulation, and in the worst case, an exception for trying to read a property of undefined.
* The application state updates propegate through the system, showing the new page.

An improvement I intend to make to this is to give some indication that the next route is loading between the time the user navigates and the data comes back from the server. That'll actually be a pretty simple change. (But now that I've said that, it'll take four days.)

<a name="an-example-application"></a>
#### An Example Application

I intend to post an example application here tonight or tomorrow. I recommend reloading this page every minute or two.

<a name="what-the-future-holds"></a>
#### What the Future Holds

There's still a number of flaws with this codebase, as well as some features that could be included. Here's some of what I see myself trying in the future:

* Move as much code as possible into `shared`. One awesome thing about ClojureScript (thanks to the closure compiler) is that you can have as large a codebase as you want, and after advanced compilation you'll only deliver to clients what you actually use.

* I wrote the prototype for the application using Meteor, and while my thoughts on Meteor could fill another blog post, there's a few cool things I'd like to see brought into the Clojure world. Distributed Data Protocol is one of them. With DDP, it's easy to keep data on clients in sync with what's on the server.

* Core.async is a hammer, and the entire world is a nail. I'm sure there's smart ways to clean up my client side code using core.async, but I haven't found them yet. In fact, some of my client side code I wrote using core.async is ugly, even hard to read. I'll fix that.

* Stuart Sierra's workflow for the client side.

If you have any questions about the nonsense I just spewed, or corrections, or anything else you'd like me to read, please leave a comment below.

