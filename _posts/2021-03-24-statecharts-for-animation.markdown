---
layout: post
title:  "Using clj-statecharts to Manage Character Animations"
date:   2021-03-24 10:00:00 -0700
categories: gamedev
---

I've been spending my spare time doing some game development in ClojureScript, and recently I needed to solve
the problem of dealing with animations for the player character. Having used the Unity game engine in the past,
I knew that finite state machines (FSM) were a great way to manage transitioning between different animations for
a character, based on underlying state changes.

For my game, I am currently dealing with 4 animations:

1. `idle` -  When standing still
2. `run` - When moving 
3. `jump` - When jumping
4. `attack` - When attacking

_These animations come from the [Kenney Character Assets bundle][kenney-characters]_

So, I spent some time building out a very poor FSM implementation for managing the animation states and
transitioning the animations when the character's state changes due to player input.

I soon learned that my FSM implementation was not going to cut it as my animation needs were a bit more advanced
than I originally thought. I found two issues:

1. The `idle`, `run`, and `jump` animations were mutually exclusive, but the `attack` animation actually needs
to be blended with the other animations.
2. For the `attack` animation, I also wanted to emit an event at a particular time in the animation to signal
that the "hit" procedure should execute, as the sword-swing doesn't really land until about 1/2 second into the animation.

I was going to start researching the different FSM libraries available for Clojure, when, literally the next day, I saw a [post on the Clojure subreddit][clojure-subreddit] about a great library called [clj-statecharts][clj-statecharts] by [Lucy Wang][lucy-wang].

After reading through the documentation, I decided to try using it to replace my terrible FSM implementation.

Here's a simplified version of what my character animation FSM looks like in clj-statecharts:

{% highlight clojure %}
(def lower-body-fsm {:initial :idle
                     :states {:idle {:on {:tick [{:target :jump
                                                  :guard airborne?}
                                                 {:target :run
                                                  :guard moving?}]}
                                     :entry #(play-animation! :character/idle)}
                              :jump {:on {:tick [{:target :run
                                                  :guard (every-pred (complement airborne?)
                                                                     moving?)}
                                                 {:target :idle
                                                  :guard (complement airborne?)}]}
                                     :entry #(play-animation! :character/jump)}
                              :run {:on {:tick [{:target :jump
                                                 :guard airborne?}
                                                {:target :idle
                                                 :guard (complement moving?)}]}
                                    :entry #(play-animation! :character/run)]}}})

(def upper-body-fsm {:initial :attack-idle
                     :states {:attack-idle {:on {:tick [{:target :attack-start
                                                         :guard attacking?}]}}
                              :attack-start {:after [{:delay 450
                                                      :target :attack-end
                                                      :actions [#(emit-event! :attack-hit)]}]
                                             :entry #(play-animation! :character/attack)}
                              :attack-end {:on {:tick [{:target :attack-idle
                                                        :guard #(animation-complete? :character/attack)}]}}}})

(def full-fsm {:id :character-animation
               :type :parallel
               :regions {:lower-body lower-body-fsm
                         :upper-body upper-body-fsm}})
                         
(defn tick! [service]
  (fsm/send service :tick)

(defn init! []
  (let [machine (fsm/machine full-fsm)
        service (fsm/service machine)]
    (fsm/start service)))
                                                        
{% endhighlight %}

A few things to note about this FSM:

* The only event ever sent is the `:tick` event, which is fired on every frame. I make extensive use of guarded transitions to determine if a transition should be made based on the underlying game-state. This is represented by all of the predicates (`airborne?`, `moving?`, `attacking?` etc)
* The undisclosed `play-animation!` function deals with playing the animation via the `AnimationMixer` in Three.js.
* The undisclosed `emit-event!` function simply emits an event to character's event-system
* I'm using parallel states of `:upper-body` and `:lower-body` to separate the animation layers. So the character can be in both a `:run` state for the `:lower-body` and `:attack-start` state for the `:upper-body`
* I'm using delayed transitions to handle emitting the event after 450ms have passed since the attack animation started.

Overall, I'm really happy with how well clj-statecharts worked for this. It feels much easier to manage than my previous hacky FSM system, and it will be a very useful tool to have for other aspects of my game.

Here's what the result looks like in-game:

![](/assets/images/clj_statecharts_animations.gif)


[clojure-subreddit]: https://www.reddit.com/r/Clojure/comments/mc8o64/parallel_states_now_supported_in_cljstatecharts/
[clj-statecharts]: https://github.com/lucywang000/clj-statecharts
[lucy-wang]: https://github.com/lucywang000
[kenney-characters]: https://kenney.itch.io/kenney-character-assets
