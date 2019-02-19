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

In your platform-specific projects you'll need a subclass of `ViewRenderer`. The one for iOS looks like this:

```
using System.ComponentModel;
using System.Linq;
using CoreGraphics;
using PreviewGridLines.Helpers;
using PreviewGridLines.iOS.Renderers;
using PreviewGridLines.Views;
using UIKit;
using Xamarin.Forms;
using Xamarin.Forms.Platform.iOS;

[assembly: ExportRenderer(typeof(PreviewGrid), typeof(PreviewGridRenderer))]

namespace PreviewGridLines.iOS.Renderers
{
    public class PreviewGridRenderer : ViewRenderer<PreviewGrid, UIView>
    {
        private bool _isShowingGridLines;
        private CGColor _gridLinesColor;
        private CGColor _gridFillColor;
        private CGColor _paddingFillColor;

        protected override void OnElementChanged(ElementChangedEventArgs<PreviewGrid> e)
        {
            base.OnElementChanged(e);

            if (e.NewElement != null)
            {
                UpdateBackgroundColor();
                UpdateShowingGridLines();
                UpdateGridLinesColor();
            }
        }

        protected override void OnElementPropertyChanged(object sender, PropertyChangedEventArgs e)
        {
            base.OnElementPropertyChanged(sender, e);

            if (e.PropertyName == VisualElement.BackgroundColorProperty.PropertyName)
            {
                UpdateBackgroundColor();
            }
            if (e.PropertyName == PreviewGrid.IsShowingGridLinesProperty.PropertyName)
            {
                UpdateShowingGridLines();
            }
            if (e.PropertyName == PreviewGrid.GridLinesColorProperty.PropertyName)
            {
                UpdateGridLinesColor();
            }
        }

        private void UpdateBackgroundColor()
        {
            if (!(Element is PreviewGrid element) || !DesignMode.IsDesignModeEnabled)
            {
                return;
            }

            Layer.BackgroundColor = element.BackgroundColor.ToCGColor();
        }

        private void UpdateShowingGridLines()
        {
            if (!(Element is PreviewGrid element) || !DesignMode.IsDesignModeEnabled)
            {
                return;
            }

            _isShowingGridLines = element.IsShowingGridLines;
            SetNeedsDisplay();
        }

        private void UpdateGridLinesColor()
        {
            if (!(Element is PreviewGrid element) || !DesignMode.IsDesignModeEnabled)
            {
                return;
            }

            var elementColor = element.GridLinesColor;

            _gridLinesColor = elementColor.ToCGColor();
            _gridFillColor = elementColor.MultiplyAlpha(0.2f).ToCGColor();
            _paddingFillColor = elementColor.MultiplyAlpha(0.1f).ToCGColor();

            SetNeedsDisplay();
        }

        public override void Draw(CGRect rect)
        {
            base.Draw(rect);

            if (!(Element is PreviewGrid element) || 
                !_isShowingGridLines ||
                !DesignMode.IsDesignModeEnabled)
            {
                return;
            }

            // Reflect on Grid object to get its calculated column widths and row heights
            var columnDefinitions = element.ColumnDefinitions;
            var columnWidths = columnDefinitions.Select(
                d => (double)ReflectionHelper.GetPrivatePropertyValue("ActualWidth", d))
                .ToList();
            var rowDefinitions = element.RowDefinitions;
            var rowHeights = rowDefinitions.Select(
                d => (double)ReflectionHelper.GetPrivatePropertyValue("ActualHeight", d))
                .ToList();

            var padding = element.Padding;
            var columnSpacing = (float)element.ColumnSpacing;
            var rowSpacing = (float)element.RowSpacing;

            var x = (float)padding.Left;
            var y = (float)padding.Top;
            var w = (float)Bounds.Width - (float)padding.Left - (float)padding.Right;
            var h = (float)Bounds.Height - (float)padding.Top - (float)padding.Bottom;

            using (var g = UIGraphics.GetCurrentContext())
            {
                g.SetLineWidth(0.5f);

                // Fill padding area
                g.SaveState();

                g.SetStrokeColor(_gridLinesColor);
                g.SetFillColor(_paddingFillColor);

                var paddingPath = new CGPath();
                paddingPath.AddRect(new CGRect(0f, 0f, Bounds.Width, Bounds.Height));
                paddingPath.AddRect(new CGRect(x, y, w, h));
                g.AddPath(paddingPath);

                g.DrawPath(CGPathDrawingMode.EOFillStroke);

                g.RestoreState();

                var gridPath = new CGPath();

                // Draw columns
                // Fill column spacing & stroke column boundaries
                var columnX = x;
                for (var column = 0; column < columnWidths.Count - 1; column++)
                {
                    columnX += (float)columnWidths[column];

                    // Fill column spacing
                    g.SaveState();

                    g.SetStrokeColor(_gridFillColor);
                    g.SetFillColor(_gridFillColor);

                    var columnSpacingPath = new CGPath();
                    columnSpacingPath.AddRect(new CGRect(columnX, y, columnSpacing, h));
                    g.AddPath(columnSpacingPath);

                    g.DrawPath(CGPathDrawingMode.FillStroke);

                    g.RestoreState();

                    // Add column divider line
                    columnX += (float)columnSpacing / 2f;
                    gridPath.MoveToPoint(columnX, y);
                    gridPath.AddLineToPoint(columnX, y + h);
                    columnX += (float)columnSpacing / 2f;
                }

                // Draw rows
                // Fill row spacing & stroke row boundaries
                var rowY = y;
                for (var row = 0; row < rowHeights.Count - 1; row++)
                {
                    rowY += (float)rowHeights[row];

                    // Fill row spacing
                    g.SaveState();

                    g.SetStrokeColor(_gridFillColor);
                    g.SetFillColor(_gridFillColor);

                    var rowSpacingPath = new CGPath();
                    rowSpacingPath.AddRect(new CGRect(x, rowY, w, rowSpacing));
                    g.AddPath(rowSpacingPath);

                    g.DrawPath(CGPathDrawingMode.FillStroke);

                    g.RestoreState();

                    // Add row divider line
                    rowY += (float)rowSpacing / 2f;
                    gridPath.MoveToPoint(x, rowY);
                    gridPath.AddLineToPoint(x + w, rowY);
                    rowY += (float)rowSpacing / 2f;
                }

                // Draw column & row dividers
                g.AddPath(gridPath);
                g.SetStrokeColor(_gridLinesColor);
                g.DrawPath(CGPathDrawingMode.Stroke);
            }
        }
    }
}
```

Here is an example of what this looks like in practice:

```
<?xml version="1.0" encoding="utf-8"?>
<ContentPage 
    xmlns="http://xamarin.com/schemas/2014/forms" 
    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml" 
    xmlns:local="clr-namespace:PreviewGridLines" 
    xmlns:views="clr-namespace:PreviewGridLines.Views" 
    x:Class="PreviewGridLines.MainPage">
    <views:PreviewGrid 
        BackgroundColor="Transparent"
        Margin="12"
        Padding="24"
        ColumnSpacing="12"
        RowSpacing="24"
        VerticalOptions="StartAndExpand" 
        IsShowingGridLines="true"
        GridLinesColor="Teal"
        >
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="*" />
            <ColumnDefinition Width="*" />
            <ColumnDefinition Width="*" />
        </Grid.ColumnDefinitions>
        <Grid.RowDefinitions>
            <RowDefinition Height="50" />
            <RowDefinition Height="Auto" />
            <RowDefinition Height="50" />
            <RowDefinition Height="*" />
        </Grid.RowDefinitions>
        <Label 
            Grid.Column="0" 
            Grid.Row="0" 
            Margin="12"
            Text="0,0" 
            TextColor="Silver"
            FontAttributes="Italic,Bold"
            BackgroundColor="Red"
            VerticalOptions="Start"
            HorizontalTextAlignment="Start"
        />
        <Label 
            Grid.Column="0"
            Grid.Row="1"
            Grid.ColumnSpan="3"
            Text="Welcome to Xamarin.Forms!" 
            HorizontalOptions="Center" 
            VerticalOptions="Center" 
        />
        <Label 
            Grid.Column="1" 
            Grid.Row="2" 
            Text="1,2" 
            VerticalOptions="Center"
            HorizontalTextAlignment="Center"
            BackgroundColor="Silver"
        />
        <Label 
            Grid.Column="2" 
            Grid.Row="3" 
            Margin="12"
            Text="2,3" 
            TextColor="White"
            FontAttributes="Bold"
            BackgroundColor="Green"
            VerticalOptions="End"
            HorizontalTextAlignment="End"
        />
    </views:PreviewGrid>
</ContentPage>
```

![Screen Shot 2019-02-19 at 14.29.41.png]({{site.baseurl}}/_drafts/Screen Shot 2019-02-19 at 14.29.41.png)




