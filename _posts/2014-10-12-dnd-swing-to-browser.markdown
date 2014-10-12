---
layout: post
title:  "Drag-and-drop from the Swing app to the browser"
categories: swing, java, html, javascript
---
I doubt that this functionality can be useful for someone in 21st century. I'm sure no one else uses Swing now :) Anyway I got a business requirement to do that. 
I've been using Swing for 3 years, not so often and didn't touch drag-and-drop in swing before. This task sounds pretty crazy at the beginning. I was sure that 
I can't communicate in the Swing app with the outside world - I was wrong. First thing which I checked, I tried to drag a folder in project tree of IDEA and 
 drop to the desktop and this worked. Next I started to google and found nothing which can help me with my task.
 
Here I started my experiments which produced following code.

To recognize drag-and-drop gestures here we have *java.awt.dnd.DragSource*. Let's assume we need to have a draggable button on a frame.

{% highlight java %}
JPanel panel = new JPanel();
JButton btn = new JButton("Draggable button");

new DragSource().createDefaultDragGestureRecognizer(
    btn,
    DnDConstants.ACTION_COPY_OR_MOVE,
    (DragGestureEvent event) -> {
        event.startDrag(DragSource.DefaultLinkDrop, new TransferData());
    });

panel.add(btn);
add(panel);
{% endhighlight %}

If you try to drag the button your cursor will be shown as a draggable action happens. *TransferData* represents the way of data transfer. And it could be
some thing like that:

{% highlight java %}
private static class TransferData implements Transferable {

    private Gson gson = new Gson();

    public DataFlavor[] getTransferDataFlavors() {
        return new DataFlavor[]{
                new DataFlavor(String.class, null)
        };
    }

    public boolean isDataFlavorSupported(DataFlavor flavor) {
        return true;
    }

    @Override
    public Object getTransferData(DataFlavor flavor) 
        throws UnsupportedFlavorException, IOException {
        return gson.toJson(Data.createRandomData());
    }
}
{% endhighlight %}

*getTransferDataFlavors* method tells *DragSource* which type of data will be transferred here.

*getTransferData* method returns the data itself. As you see I used [Gson](https://code.google.com/p/google-gson/) to serialize my business object to json.

Now we need to introduce the receiving side. It'll be a plain html page. The most interesting part of it is javascript code. 
There we need to define listeners for for 3 events: *dragover, dragenter, drop*. It's important to call *preventDefault* and *stopPropagation* to prevent 
others actions and make this code work in all browsers, [Here](http://stackoverflow.com/questions/20354439/html5-drag-drop-e-stoppropagation) is briefly explanation why we need to do that.

{% highlight javascript %}
$('#dropArea').on('dragover',
        function (e) {
            e.preventDefault();
            e.stopPropagation();
        }
).on('dragenter',
        function (e) {
            e.preventDefault();
            e.stopPropagation();
        }
).on('drop',
        function(e){
            if(e.originalEvent.dataTransfer){
                e.preventDefault();
                e.stopPropagation();
                
                var data = JSON.parse(
                    e.originalEvent.dataTransfer.getData('text')
                );
                //your stuff goes here
            }
        }
);
{% endhighlight %}

Here we go. The working example you can find in my [github project with examples](https://github.com/dimafeng/dimafeng-examples/tree/master/drag-and-drop). 