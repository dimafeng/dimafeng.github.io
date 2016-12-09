---
layout: note
title:  "Drag-and-drop from the Swing application. Files"
categories: swing, java
---
Here the continuation of the previous article about [dragging and dropping from the swing app](/notes/2014/10/12/dnd-swing-to-browser/). Recently I got requirement to implement the drag-and-drop for reports in our swing application as files into any system object which able to be 'dropable' (it could be exprorer in windows, finder in macos, etc.). Also, this functionality should help implement correct reports uploading into rich editor (like [redactor.js](http://imperavi.com/redactor/)) in the browser. Few words about *redactor*: it's javascript rich text editor. The editor is pretty small and flexable. And it has files/images uploading by DnG gesture out the box. We spended a few hours to configure our java based backend to work as a storage for this editor. We've got here is ability to drag report in swing app and drop it in system's folder or in the text editor in browser.

Now I'm going to introduce small snippet which you can use in your swing app.

{% highlight java %}
private static class TransferFileData implements Transferable
{
    private final PrintableFrame printableFrame;

    private TransferFileData(PrintableFrame printableFrame)
    {
        this.printableFrame = printableFrame;
    }


    @Override
    public Object getTransferData(DataFlavor flavor)
    throws UnsupportedFlavorException, IOException
    {
        if (isDataFlavorSupported(flavor))
        {
            Path path = Files.createTempFile(printableFrame.getDefaultFilename(), new Date().getTime() + ".pdf");
            printableFrame.saveContent(path.toFile(), FileTypes.PDF);
            return Arrays.asList(path.toFile());
        }
        else
        {
            return null;
        }
    }


    @Override
    public DataFlavor[] getTransferDataFlavors()
    {
    return new DataFlavor[] {DataFlavor.javaFileListFlavor};
    }


    @Override
    public boolean isDataFlavorSupported(DataFlavor flavor)
    {
        return DataFlavor.javaFileListFlavor.equals(flavor);
    }
}
{% endhighlight %}

*PrintableFrame* is our internal interface which can be exported in pdf. As you notice we're generating pdf reports on the fly. All files will be stored in temp directory. If you need to do a lot of such drag-and-drop actions you have to think about deletion this temp stuff.
