---
published: false
---
## Showing Gridlines In Xamarin Forms Previewer

This blog post describes how to show make a Xamarin.Forms `Grid` show its columns and rows graphically in the Visual Studio XAML Preview.

One of the things about the Previewer that has always frustrated me is that it doesn't show the outlines of any of the objects that I place in a form.

A simple way to visualise the grid is to create some `BoxView` objects, give them `BackgroundColor`, and add them to the `Grid`. However, it's easy to see that this is ugly, prone to errors (I have accidentally shipped something with a hot pink background at least once) and extremely tedious.

Xamarin.Forms objects are cross-platform virtualisations that draw on each platform by means of a Renderer. Delving into the source of Xamarin Forms revealed that the `Grid` object is a subclass of `Layout<View>`, but unusually it has no renderer of its own. This is because all it does is manage a collection of child views and arrange them grid-fashion in its parent view.

Most Xamarin.Forms objects are open for subclassing. The `Grid` object definitely is, so we can subclass it in our platform-independent code to make a `PreviewGrid`:

```
using Xamarin.Forms;

namespace PreviewGridLines.Views
{
    public class PreviewGrid : Grid
    {
        public static readonly BindableProperty IsShowingGridLinesProperty =
            BindableProperty.Create(
                nameof(IsShowingGridLines),
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

        public bool IsShowingGridLines
        {
            get => (bool)GetValue(IsShowingGridLinesProperty);
            set => SetValue(IsShowingGridLinesProperty, value);
        }

        public Color GridLinesColor
        {
            get => (Color)GetValue(GridLinesColorProperty);
            set => SetValue(GridLinesColorProperty, value);
        }
    }
}
```

In your platform-specific projects you'll need a subclass of `ViewRenderer`. Here's one for iOS:

```
PreviewGridRenderer
```

I've left out most of the boilerplate code. If you want to see the whole example, it's available on my GitHub page.
