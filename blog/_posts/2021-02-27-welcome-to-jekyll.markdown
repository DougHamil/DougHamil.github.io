---
layout: post
title:  "Autotyping-Text for Game Dialogue with Reagent"
date:   2021-02-27 10:48:00 -0700
categories: gamedev clojurescript reagent
---
_Thanks for reading my first blog post!_

One of my favorite hobbies is developing video games and one of my favorite programming languages is Clojure. 
I have tried to combine these two interests of mine in different ways. One of my favorite personal projects was building a ClojureScript library called [Threeagent][threeagent] which can be used to build
3D browser-based applications in a very [Reagent][reagent]-esque way. I gave a [live-coding demonstration][threeagent-talk] of how to use the library, if you're interested in seeing it in action.

I recently started yet another personal game development project (which, more likely than not, will end up in my pile of unfinished projects) and I've decided to blog about any interesting details that come up as I continue my work on this project.
My new project is going to be a browser-based game, which will use Reagent for the UI and Threeagent for the 3D scenes.

For my first post, I want to review how I implemented a very common component: autotyping-text. Autotyping-text is often used in video games without voice-acting to make the
text-based dialogue more dynamic and interesting to the player. We will be using Reagent to implement this component and I will provide and walk-through the code of the implementation.

![](https://www.earthboundtext.com/images/splash.gif)

*An example of auto-typing text (https://www.eartboundtext.com)*

For my autotyping component, I want to render a text string that will have the typing animation last some specified duration.
For the animation, I'll start with defining a lerp function that returns a reactive atom that my component can reference to determine how much of the text to display:

{% highlight clojure %}
(ns app.interp
  (:require [threeagent.core :as th]
            [app.math :as math]
            ["uuid" :as uuid]))
             
(defonce ^:private interp-atoms (atom {}))

(defn halt-atom! [a]
  "Unregister the watch-fn for a given lerp atom from the associated timer-derefable"
  (when-let [{:keys [key timer-atom]} (get @interp-atoms a)]
    (remove-watch timer-atom key)
    (swap! interp-atoms dissoc a)))

(defn lerp-atom
  "Given a deref-able timer atom/cursor and a duration, return a reactive-atom with the current lerp value"
  ([timer-atom duration resolution]
   (let [start-time @timer-atom
         key (uuid/v4)
         this-atom (th/atom 0.0)]
     ;; Watch timer and update lerp
     (add-watch timer-atom key (fn [_ new-time]
                                 (let [d (- @new-time start-time)
                                       t (math/clamp01 (/ d duration))]
                                   (if resolution
                                     (reset! this-atom (.toFixed t resolution))
                                     (reset! this-atom t)))))
     (swap! interp-atoms assoc this-atom {:key key
                                          :timer-atom timer-atom})
     this-atom))
  ([timer-atom duration]
   (lerp-atom timer-atom duration nil)))
{% endhighlight %}

With this code, we can call the `lerp-atom` function with an atom/cursor of the current time and a duration, and get back a reactive-atom that we can deref to get the current lerp value (0.0 to 1.0).
The `lerp-atom` function works by adding a watch to the provided timer atom/cursor and updating the reactive-atom with the newly calculated lerp-value.  The `resolution` argument can be used to set a specific precision for the lerp value, this can help reduce unnecessary re-renders of our Reagent components when they don't need high precision.

Because we are using the `add-watch` function, we need a way to remove our watch-fn from the timer-atom whenever a component is unmounted from our scene. The `halt-atom!` function handles this by looking up the associated timer-atom of a given lerp-atom and using the `remove-watch` function to unregister our watch-fn for this particular lerp-atom. We use the uuid library as a way to generate unique keys for each lerp-atom we create.

With our reactive lerp-atom ready to go, we can move on to defining our Reagent component to render the text:

{% highlight clojure %}
(defn dialogue-box [{:keys [duration text]}]
  (let [timer (ent/cursor state [:time]) ;; Use entanglement cursor for add-watch support
        lerp (interp/lerp-atom timer duration 2)
        text-length (count text)]
    (r/create-class
     {:display-name "dialogue-box"
      :component-will-unmount #(interp/halt-atom! lerp)
      :reagent-render (fn [_]
                        (let [text (take (js/Math.floor (* text-length @lerp))
                                         text)]
                          [:div text]))})))
{% endhighlight %}

Our `dialogue-box` component is defined using Reagent's `create-class` function because we need to add a hook
into the component's lifecycle so we can call the `halt-atom!` function when our component is unmounted.

Our component works by first getting a cursor to a `:time` field in our global state atom. In this case I had to use the [Entanglement][entanglement] library's cursor function because using `add-watch` on a [standard Reagent cursor won't work because the cursors are lazy][reagent-cursor-addwatch], so we would have to explicitly deref a Reagent cursor in our component.

Other than that, our component is pretty straightforward! We simple `take` a percentage of characters from our original text using our lerp value and render it out as a `:div`. Notice in this example that we are using a resolution of 2, because we don't need any more than 2 decimals of precision on the lerp value to get good looking results.

We can now add our `dialogue-box` component to our UI and the result will look like this:

![](/assets/images/autotype_result.gif)

And that's it! Not only do we have a Reagent-friendly way of animating text in this way, we also have established a pattern for handling interpolation and reactively re-rendering our components based on the interpolated values.


[reagent]: https://github.com/reagent-project/reagent
[threeagent]: https://github.com/DougHamil/threeagent
[threeagent-talk]: https://www.youtube.com/watch?v=myigRnZHhTw
[entanglement]: https://github.com/Frozenlock/entanglement
[reagent-cursor-addwatch]: https://github.com/reagent-project/reagent/issues/244
