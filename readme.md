- [Svelte Crossfade](#svelte-crossfade)
    - [Examples](#examples)


## Svelte Crossfade

A common issue when trying to transition between two distinct layouts, animating the elements that remain betweeen both component states is that there is often layout competition issue.

This is most common when structure A is being replaced with structure B. Where A and B share a structure C. While the animation exists fading out and B in, the layout is disrupted and results in awkward offsets and abrupt jumps in element positions. 

One solution to deal with this particular problem is to use the grid layout to force the various components that will occupy the same space into the same cell.

#### Examples
* [Tab Navigation with animated crossfade div transition](https://svelte.dev/repl/efbe0f92d43a403ea2e5ac6251a55f18?version=3.29.7)

* [Notification Stack - if/else](https://svelte.dev/repl/92d5891ad4324437aa77fd4574ce4e2e?version=3.29.7)