---
published: false
---
## Showing Gridlines In Xamarin Forms Previewer

I do a lot of work with Xamarin Forms. I'm in the XAML Previewer for much of my working day as I create new features and fix bugs in our company's app.

One of the things about the Previewer that has always frustrated me is that it doesn't show the outlines of any of the objects that I place in a form.

The standard tactic to get around this limitation is to apply a `BackgroundColor` to each object being added. However, it's easy to see that this is ugly, prone to errors (I have accidentally shipped something with a `Fuchsia` (hot pink) background at least once) and extremely tedious.

I began to be convinced that "there must be a better way".