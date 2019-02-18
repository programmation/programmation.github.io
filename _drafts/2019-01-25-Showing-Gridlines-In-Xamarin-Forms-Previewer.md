---
published: false
---
## Showing Gridlines In Xamarin Forms Previewer

I do a lot of work with Xamarin Forms. I'm in the XAML Previewer for much of my working day as I create new features and fix bugs in our company's app.

One of the things about the Previewer that has always frustrated me is that it doesn't show the outlines of any of the objects that I place in a form.

The standard tactic to get around this limitation is to apply a `BackgroundColor` to each object being added. However, it's easy to see that this is ugly, prone to errors (I have accidentally shipped something with a `Fuchsia` (hot pink) background at least once) and extremely tedious.

After thinking about it for a while, I began to be convinced that "there must be a better way".

Initially a custom renderer looked like a good idea. Delving into the source of Xamarin Forms revealed that the `Grid` object is a subclass of `Layout<View>`, but it has no renderer of its own - all it does is manage a collection of child views and arrange them grid-fashion in its parent view.

The next step was to subclass the `Grid` to `PreviewGrid` and create the custom `PreviewGridRenderer`:

```
PreviewGrid
```

```
PreviewGridRenderer
```

I've left out most of the boilerplate code. If you want to see the whole example, it's available on my GitHub page.
