---
published: false
---
## Showing Gridlines In Xamarin Forms Previewer

I do a lot of work with Xamarin Forms. I'm in the XAML Previewer for much of my working day as I create new features and fix bugs in our company's app.

One of the things about the Previewer that has always frustrated me is that it doesn't show the outlines of any of the objects that I place in a form.

One way to visualise the grid is to apply a `BackgroundColor` to each object being added. However, it's easy to see that this is ugly, prone to errors (I have accidentally shipped something with a `Fuchsia` (hot pink) background at least once) and extremely tedious.

The search for a solution led initially to a custom renderer. Delving into the source of Xamarin Forms revealed that the `Grid` object is a subclass of `Layout<View>`, but it has no renderer of its own - all it does is manage a collection of child views and arrange them grid-fashion in its parent view.

To create the custom renderer, subclass `Grid` in your platform-independent code to make a `PreviewGrid`:

```
using Xamarin.Forms;

namespace PreviewGridLines.Views
{
    public class PreviewGrid : Grid
    {
        public static readonly BindableProperty ShowGridLinesProperty =
            BindableProperty.Create(
                nameof(ShowGridLines),
                typeof(bool),
                typeof(PreviewGrid),
                false
            );

        public static readonly BindableProperty GridLinesColorProperty =
            BindableProperty.Create(
                nameof(GridLinesColor),
                typeof(Color),
                typeof(PreviewGrid),
                Color.Default
            );

        public bool ShowGridLines
        {
            get => (bool)GetValue(ShowGridLinesProperty);
            set => SetValue(ShowGridLinesProperty, value);
        }

        public Color GridLinesColor
        {
            get => (Color)GetValue(GridLinesColorProperty);
            set => SetValue(GridLinesColorProperty, value);
        }

        public PreviewGrid()
        {
        }
    }
}
```

In your platform-specific projects you'll need a subclass of `ViewRenderer`. Here's one for iOS:

```
PreviewGridRenderer
```

I've left out most of the boilerplate code. If you want to see the whole example, it's available on my GitHub page.
