---
layout: post
title: "Do Me a Favor When X happens! Making a code Programmable Using Dynamic Callbacks"
categories: [software architecture]
tags: ["state machines", "C", "FreeRTOS"]
---

**TL;DR**
While implementing a custom networking protocol for an IoT project, I ran into a decoupling problem: a central event emitter needed to reset timers owned by separate state machines. Hardcoding this logic into the emitter created a tight coupling anti pattern. I solved this by implementing a thread safe, dynamic callback system in C, complete with hook points and self unregistering callback actions.

## Context: The "Silly Proto"
For my IoT internship, I needed to implement a networking protocol, this blog is about solving a decoupling problem using hook points and dynamic callbacks, I named it silly proto (the name stuck after some iterations on a really silly protocol).

The Implementation's entities which we are interested in this blog are (each one is running in a separate FreeRTOS task):
- Emitter: takes input from other tasks, publishes events based on the input to consumer state machines
- State machines: consume events and carry out what they need to

![](/assets/svg/do_me_a_favor_when_x_happens/intro_to_emitter.svg)

## The Problem: Who Should be Responsible for This?

Emitter's job is simple and its happy doing it's job in peace, until the protocol introduces timers that need resetting when an event happens (especially when the timer's owning state machine isn't interested in the event).
One concrete example of this situation is the following timers (don't pay attention to the name of the state machine, the role of the state machines are irrelevant in this blog):

```c
struct silly_master_state_machine {
	// --- out of blog's scope fields ---
	
    /**
     * master_death_timeout
     * - initiate: entry of `MASTER_MODE::STAND_DOWN`
     * - reset: recv in the emitter task
     * - delete: exit of `MASTER_MODE::STAND_DOWN`
     */
    esp_timer_handle_t master_death_timeout;
    SemaphoreHandle_t master_death_timeout_mu;

    /**
     * network_stop_timeout
     * - initiate: entry of `MASTER_MODE::WATCH_NETWORK`
     * - reset: recv in the emitter task
     * - delete: exit of `MASTER_MODE::WATCH_NETWORK`
     */
    esp_timer_handle_t network_stop_timeout;
    SemaphoreHandle_t network_stop_timeout_mu;
    
    // --- out of blog's scope fields ---
};

```

All timers have their own values, and they send their unique values to emitter when their expiration callback is called, so the emitter can emit a time out event.

The question is, where do we reset the timers? Do we subscribe to the RECV_ANY (is emitted when anything is received, corrupted or healthy packets) event for the sole purpose of resetting the timer? 
We should let the emitter reset the timer so we have more chance of not letting the timer expire due to scheduling delays. But putting this logic in the emitter isn't scalable, we will find ourselves putting any out of scope logic in emitter later on. So, we got a coupling problem with the idea of emitter doing something that is specific to it's clients and not necessarily its concern.

![](/assets/svg/do_me_a_favor_when_x_happens/emitter_being_sad_too_much_responsibility.svg)

## The Solution: Hook Points and Dynamic Callbacks
A great way to solve this is implementing a dynamic callback system. We introduce hook points in the emitter, and on each of those hook points, we call the the registered (if present) callbacks.
Hook points:
```c
enum EMITTER_HOOKPOINT {
    _EMITTER_HOOKPOINT_UNSPEC,
    EMITTER_HOOKPOINT_ON_RECV,
    _EMITTER_HOOKPOINT_SIZE,
};
```

An example of a hook point triggering:
```c
// is called on processing a received packet
int receiver_elem_process(
    struct emitter_task *self, void **vreciever_elem)
{
	// some code
	
    trigger_hook_point(self, EMITTER_HOOKPOINT_ON_RECV);
	
	// some code
}
```

What if we register a callback that is meant to run only once the callback can't unregister itself because it doesn't have access to the callback linked list (technically it does, but it would be a complicated lookup through `self`)? I'm pleased to introduce to you, callback actions (inspired by eBPF's XDP and LSM)! If a callback is done and wants to be removed, it can return the related action, and the `trigger_hook_point` will unregister the callback.

```c
enum EMITTER_CALLBACK_ACTION {
    _EMITTER_CALLBACK_ACTION_UNSPEC,
    EMITTER_CALLBACK_ACTION_NOOP,
    EMITTER_CALLBACK_ACTION_DELETE_CALLBACK,
    _EMITTER_CALLBACK_ACTION_SIZE,
};
```

The callback struct and registration function:
```c
struct emitter_callback {
    enum EMITTER_CALLBACK_ACTION (*callback)(
        struct emitter_task *self, void *);
    void *argument;
    void (*free_arguement)(void *);
};

/**
 * @brief add the callback to emitter task
 * @note can block
 * can be used by other tasks at emitter's run time
 */
int add_callback_to_emitter(
    struct emitter_task *self,
    struct emitter_callback **callback,
    enum EMITTER_HOOKPOINT hookpoint);

```

What about concurrency? how do we keep the callback calling and registration thread safe? We add a lock per event (because we maintain a linked list of callbacks per event):
```c
struct emitter_task {
		
    struct linked_list_node *emitter_callbacks[_EMITTER_HOOKPOINT_SIZE];
    SemaphoreHandle_t emitter_callbacks_mu[_EMITTER_HOOKPOINT_SIZE];
	
	// out of blog's scope fields
};

```

A little peek into `trigger_hook_point` (code is shortened for better readability):
```c
static void trigger_hook_point(
    struct emitter_task *self, enum EMITTER_HOOKPOINT hook_point)
{
	// aquires the semaphore here, releases when getting out of scope
    SCOPED_SEMAPHORE_TAKE(
        self->emitter_callbacks_mu[hook_point], portMAX_DELAY, lock_acquired);

	// executes the following code block on each node in the linked list (head being the third arguement)
	// LINKED_LIST_FOREACH is resolving the next_node because the current node might get freed and unlinked in the middle of iteration
    LINKED_LIST_FOREACH(
        current_node, next_node, self->emitter_callbacks[hook_point])
    {
        current_callback = current_node->data;
        callback_action =
            current_callback->callback(self, current_callback->argument);
			
        switch(callback_action) {
        case EMITTER_CALLBACK_ACTION_DELETE_CALLBACK: {
            linked_list_remove_node(&current_node);
            break;
        }
        case EMITTER_CALLBACK_ACTION_NOOP: {
            break;
        }
        }
    }
}
```

![](/assets/svg/do_me_a_favor_when_x_happens/state_machine_asking_for_a_favor.svg)

We can finally register our callback for resetting our timer!
```c

struct emitter_callback *reset_network_stop_timeout_emitter_callback = 
	calloc(1, sizeof(struct emitter_callback));

reset_network_stop_timeout_emitter_callback->argument =
	self; // struct silly_master_state_machine *
reset_network_stop_timeout_emitter_callback->callback =
	reset_network_stop_timeout;

add_callback_to_emitter(
    self->parent->emitter, // we haven't addressed this coupling, we may solve it later
    &reset_network_stop_timeout_emitter_callback,
    EMITTER_HOOKPOINT_ON_RECV);
```

## Caveats
- If the callback tries to register another callback for the same hook point, we encounter a recursive lock acquisition and a dead lock would happen.
