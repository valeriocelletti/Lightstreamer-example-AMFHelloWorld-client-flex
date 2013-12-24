# Lightstreamer - "Hello World" Tutorial - Flex (AMF) Client  #
<!-- START DESCRIPTION lightstreamer-example-amfhelloworld-client-flex -->

This project, of the "Hello World with Lightstreamer" series, will focus on a new feature that was [released](http://cometdaily.com/2010/02/22/lightstreamer-36-released/) with [Lightstreamer Server](http://www.lightstreamer.com/download.htm) since version 3.6: <b>Action Message Format (AMF)</b> support for Flex applications.

First, a quick recap of the previous installments:

- "Hello World" with Lightstreamer: An introduction to Lightstreamer's data model, JavaScript Client API (see  [Lightstreamer - "Hello World" Tutorial - HTML Client](https://github.com/Weswit/Lightstreamer-example-HelloWorld-client-javascript)), and Java Data Adapters API (see [Lightstreamer - "Hello World" Tutorial - Java Adapter](https://github.com/Weswit/Lightstreamer-example-HelloWorld-adapter-java)).
- [Lightstreamer - "Hello World" Tutorial - .NET Adapter](https://github.com/Weswit/Lightstreamer-example-HelloWorld-adapter-dotnet): The .NET API version of the Data Adapter used in the "Hello World" application, showing both a C# and a Visual Basic port.
- [Lightstreamer - "Hello World" Tutorial - TCP Sockets Adapter](http://cometdaily.com/2008/07/29/hello-world-for-sockets-with-lightstreamer/): The TCP-socket-based version of the Data Adapter, suitable for implementation in other languages (PHP, Python, Perl, etc).

Basically, Lightstreamer Server can be seen as a "technology hub" for data push, where you can mix different technologies on the client-side and on the server-side to exchange real-time messages.<br>
![Schema](technology-hub1.png)

We will delve into the "Flex on the client-side, Java on the server-side" scenario and, in this project, full details for the client-side will be provided.<br>

In particular, it will be shown you how to push binary ActionScript objects directly to your Flex application in real-time, using AMF. [Action Message Format](http://en.wikipedia.org/wiki/Action_Message_Format) is used for serializing binary ActionScript objects in order to transmit them over the wire.<br>

As you may recall form the [first installment](https://github.com/Weswit/Lightstreamer-example-HelloWorld-client-javascript), Lightstreamer data push model is based on items made up of fields. A client can subscribe to many items, each with its own schema (a set of fields). Each field is usually a text string with an arbitrary length (from a few characters to large data structures, perhaps based on XML, JSON, etc.). With this new Lightstreamer feature, you can now put a full AMF object into any field and have it pushed from the server to the client in real binary format (with no re-encondings).<br>

The Flex Client library for Lightstreamer has been used for one of the <b>major dealing platforms</b> in the finance field and has undergone many cycles of improvements to make it completely production resilient. We were asked to add native support for AMF objects to improve the performance when streaming complex data structures. So now you can push both text-based items and object-based items to the same Flex application.<br>

When approaching the Lightstreamer <b>data model</b>, it is important to choose the right trade-off between using <i>fine-grained fields and coarse objects</i>. You could map each individual atomic piece of information to a Lightstreamer field, thus using many fields and items, or you could map all your data to a single field of a single item. This applies to both text-based fields (where you can encode coarse objects via JSON, XML, etc.) and object-based fields (via AMF). Usually, going for fine-grained fields is better, because you let Lightstreamer know more about your data structure, so that it will be able to apply optimization mechanisms, like conflation and delta delivery. On the other hand, if you go for huge opaque objects, Lightstreamer will be used more as a blind pipe. But in both cases, you will still benefit from other features, like bandwidth allocation and dynamic throttling. All intermediate scenarios are possible too.<br>
<!-- END DESCRIPTION lightstreamer-example-amfhelloworld-client-flex -->

In the sections below, it will be used a single field containing an AMF object derived from a JavaBean. To encode a JavaBean as an AMF object, several ready-made libraries exist. Here we will leverage BlazeDS.<br>

## AMF Lightstreamer Tutorial ##

This project focuses on a simple "Hello World" example to show how to use AMF with our new Flex client library ([docs](http://www.lightstreamer.com/docs/client_flashflexas_asdoc/index.html)). We will create a JavaBean on the server-side and then use it on the client-side.

For this tutorial, I'm assuming you have already read the Basic Hello World example, or that you are already familiar with Lightstreamer concepts.

On the client, the result of this tutorial will be quite similar to the one obtained with the original [Lightstreamer - "Hello World" Tutorial - HTML Client](https://github.com/Weswit/Lightstreamer-example-HelloWorld-client-javascript), but in Flex: we'll get a string alternating some different values (Hello AMF World) and a timestamp. On the server-side, data will be encapsulated into a <b>JavaBean</b> containing a String and a Date instance. This bean will be translated into a byte array and then injected into the Lightstreamer kernel as a single field, instead of being spread over different fields as simple strings (as the original adapter does). Here lies the power of AMF, as you will be able to push even complex JavaBeans to your Flex clients with ease.

## Gather Stuff ##

- First of all you'll need a browser, a Flash player, and a JDK: hopefully you already have those :).
- You'll need <b>Lightstreamer Server (Presto or Vivace)</b> and <b>Lightstreamer Flex Client 2.0</b>. You can [download them from the Lightstreamer web site](http://www.lightstreamer.com/download.htm).
- You'll have to compile a Flex application, so you'll need either the [Flex Builder](http://www.adobe.com/mena/products/flex/) or the [Flex SDK](http://opensource.adobe.com/wiki/display/flexsdk/Flex+SDK) (the example works with Flex 3 and Flex 4; use Flex 4 only if you're going to use it with the Flex Builder, otherwise you may have problems with the mxmlc.
- The conversion from Java beans to an AMF-compatible byte array is performed by a couple of the [BlazeDS libraries](http://opensource.adobe.com/wiki/display/blazeds/Release+Builds). Download BlazeDS (binary distribution) and extract flex-messaging-common.jar and flex-messaging-core.jar from it (the downloaded zip contains a .war file, open it with an archive manager and locate the needed libraries under "WEB-INF/lib/").

## Let's Code the Front-End(s) ##

The front-end of our application will be a simple .mxml file compiled into a .swf. Open your text editor and start writing a mxml file (notice the creationComplete event):

```html
<?xml version="1.0" encoding="utf-8"?>
<mx:Application xmlns:mx="http://www.adobe.com/2006/mxml"
   creationComplete="init()">
```

Then add a TextArea instance (myTextArea), where we'll put the updates:

```html
<mx:TextArea width="693" height="56" id="myTextArea"
   fontSize="25" fontWeight="bold" text="loading..."/>
```

Now to Actionscript... We need an init method to initialize our client and subscribe to our table. We'll also configure a DateFormatter instance to print nice-looking dates.

```actionscript
public function init():void {
  dateFormatter.formatString = "MM/DD/YY J:NN:SS"; 
 
  var cInfo:ConnectionInfo = new ConnectionInfo();
  cInfo.server = "localhost";
  cInfo.adapterSet = "AMFHELLOWORLD";
 
  var client:LSClient = new LSClient();
 
  var nonVisualTable:NonVisualTable =
     new NonVisualTable(["greetings"],["AMF_field"],"MERGE",true);
  nonVisualTable.addEventListener(
    NonVisualItemUpdateEvent.NON_VISUAL_ITEM_UPDATE,onChange);
  client.subscribeTable(nonVisualTable);
 
  client.openConnection(cInfo);
}
```

The <i>ConnectionInfo</i> instance defines how we connect to the server and the adapter set that will handle our requests. In this case we'll deploy our application on tje Lightstreamer internal web server so the server has to be set to "localhost". The adapter set in use will be called "AMFHELLOWORLD".

The <i>LSClient</i> instance is the core of the Lightstreamer Flex client library. It handles the connection to the server and the subscription/unsubscription of tables.

The <i>NonVisualTable</i> represents the subscription we're going to make. It is a mono-item ("greetings") and mono-field ("AMF_field") <i>MERGE subscription</i>. The fourth parameter in the constructors tells the server that we're looking for binary data, namely AMF.

Next, we need the <i>onChange</i> callback to handle the updates and show them on screen. We'll just extract the single field and, since it's a Bean, expand it, format the date an put everything on the TextArea previously created:

```actionscript
public function onChange(evt:NonVisualItemUpdateEvent):void {
  if (evt.isFieldChanged("AMF_field")) {
    var obj:* = evt.getFieldValue("AMF_field");
    myTextArea.text = dateFormatter.format(obj.now) + " " + obj.hello;
  }
}
```

Download the complete source of the Flex client from the "AMFHelloWorld.mxml" source file of this project.<br>

If you prefer a DataGrid in place of the TextArea, don't worry, the substitution is really easy. First we replace the TextArea with a DataGrid (myDataGrid) with two columns. Each column is associated to a property of the bean sent by the server, so "AMF_field" represents the entire bean (the field of our subscription) and "AMF_field.hello" represents the "hello" property of the bean. We add a labelFunction to one of the columns so that we'll be able to format the received date.

```actionscript
<mx:DataGrid width="479" x="447.5" height="91" id="helloView" y="12" >
  <mx:columns>
    <mx:DataGridColumn dataField="AMF_field.now" 
      headerText="Time" labelFunction="formatDate" />
    <mx:DataGridColumn dataField="AMF_field.hello" headerText="Hello"/>
  </mx:columns>
</mx:DataGrid>
```

Then, we need something to be bound with the DataGrid. Luckily enough, our VisualTable class extends the [ArrayCollection](http://livedocs.adobe.com/flex/3/langref/mx/collections/ArrayCollection.html) native class. so that it can be bound to a DataGrid to update the view automagically. We have to put our VisualTable in the global space so that we can declare the [Bindable] metadata on it.

```actionscript
[Bindable] public var myTable:VisualTable;
```

Finally, we have to initialize this table. The code is very similar to the one used to initialize the NonVisualTable, but we don't need to add any event handlers. Instead, we bind our VisualTable and the DataGrid assigning our Bindable VisualTable to the "dataProvider" property of the DataGrid.

```actionscript
var myTable:VisualTable =
  new VisualTable(["greetings"],["AMF_field"],"MERGE",true);
helloView.dataProvider = myTable;
```

The complete modified source of the Flex client is shown in the "AMFHelloWorld_DataGrid.mxml" source file of this project.<br>

The mxml file, as is, is quite useless, so, save it as "AMFHelloWorld.mxml" and compile it into a swf file using the Flex SDK:

```actionscript
FLEX_SDK_HOME/bin/mxmlc AMFHelloWorld.mxml 
  -output AMFHelloWorld.swf 
  -library-path+=
    LS_HOME/DOCS-SDKs/sdk_client_flash_flex(native_as)
    /lib/Lightstreamer_as_client.swc
```
    
Note that if you're using a 64-bit JVM you may have some issues running mxmlc; use a 32-bit JVM (mxmlc makes use of the JAVA_HOME environment variable to choose the JVM).

## Deploy the Client ##

The client is made up only by the compiled swf file, so grab that file and put it under the "LS_HOME/pages" folder, that's all.

## Run! ##

Start the Lightstreamer server, please before make sure you have deployed the "AMFHELLOWORLD" Adapter having followed the steps in [this tutorial](), and point your browser to: http://localhost:8080/AMFHelloWorld.swf
Enjoy!

## Final Notes ##

You've seen how to push Objects instead of Strings from a Lightstreamer server to a Flex client. You can exploit this technique to push complex data structures, but obviously, doing so you'll lose some of the optimizations offered by Lightstreamer protocol. For example, the merging algorithm (of the MERGE mode) is applied to the entire bean instead of being applied to each single field, so that every time a property within the bean changes, the entire bean is pushed to the client, not only the changed value. As with anything regarding engineering you'll have to choose the trade-off that optimizes properly for your application.

Please also consider that the Flex client library in this tutorial is not available with the Moderato version of Lightstreamer Server, so you may want to use a [DEMO license](http://www.lightstreamer.com/download) for Lightstreamer Presto/Vivace to experiment with it.

# See Also #

## Lightstreamer Adapters needed by this demo client ##
<!-- START RELATED_ENTRIES -->

* [Lightstreamer - "Hello World" Tutorial - Java SE (AMF) Adapter](https://github.com/Weswit/Lightstreamer-example-AMFHelloWorld-adapter-java)

<!-- END RELATED_ENTRIES -->

## Related projects ##

* [Lightstreamer - "Hello World" Tutorial - HTML Client](https://github.com/Weswit/Lightstreamer-example-HelloWorld-client-javascript)
* [Lightstreamer - "Hello World" Tutorial - Java Adapter](https://github.com/Weswit/Lightstreamer-example-HelloWorld-adapter-java)
* [Lightstreamer - "Hello World" Tutorial - .NET Adapter](https://github.com/Weswit/Lightstreamer-example-HelloWorld-adapter-dotnet)

# Lightstreamer Compatibility Notes #

- Compatible with Lightstreamer Flex Client Library version 2.0 or newer.
- For Lightstreamer Allegro (+ Flex Client API support), Presto, Vivace.
