---
title: Tomcat Architecture
date: 2019-07-16 21:06:12
tags:
- Java
- Web
categories: Web
---

# 1. A heigh-level Architecture

One basic functionality of the server is receiving requests from client, analyzing the requests, running busines logic and returning the result to the client.

{% asset_img simple-server.png simple server %}

start() method starts server, opens socket, listens on port number, processes requests and responds to client. And stop() method stops the server and releses the resources.
<!--more-->
## Componets

### Connector and Container

We realize that the scalalibility is not good if we put listening functionality and processing logic together. For example, we want to support mutliple network protocol but the processing logic is identical. Tomcat supports either HTTP protocol or AJP.

Now we separate network protocol and business logic.

{% asset_img server2.png %}

One server can include multiple Connector and Container. Connector listens on requests and responses data. Container takes care of processing requests. 

There is one disadvantage of this design. How do we map connector to container.

{% asset_img server3.png %}

One Server includes multiple Service. A Service associates one or more Connectors to a Engine. Thus, requests comming to the Connectors will be handled by the Container.

We rename Containe to Engine.

### Container design

We have decoupled network protocol and container. Application server is used to deploy Web application. So Engine needs to manage multiple web application. When it receives a request, it needs to find an appropriate web application to handle this request.

{% asset_img server4.png simple server 4 %}

We use Context to represent a Web application. One Engine can include multiple Context.

Next, we want to support virtual host. So we add Host between Engine and Context.

As we know that one Web application can contain mutltiple Servlet instances. So, we add Wrapper to represent servlet.

{% asset_img server5.png simple server 5 %}

Tomcat refers to Engine, Host, Context, and Cluster, as container. The highest-level is Engine; while the lowest-level is Context. Certain components, such as Realm and Valve, can be placed in a container.

{% asset_img server6.png simple server 6 %}

### Init Flow

#### Bootstrap
Bootstrap.init() method: Initalize class loader and create Catalina object and assign it to a class private attribute
```java
private Object catalinaDaemon
//...
initClassLoaders();
Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
Object startupInstance = startupClass.getConstructor().newInstance();
//...
catalinaDaemon = startupInstance;
```

```java
 ...if (command.equals("start")) {
                daemon.setAwait(true);
                daemon.load(args);
                daemon.start();
                if (null == daemon.getServer()) {
                    System.exit(1);
                }
```

Bootstrap.load(args): Use reflection to call load method of Catalina object. Catalina object is created in init method.

```java
String methodName = "load";
Method method = catalinaDaemon.getClass().getMethod(methodName, paramTypes);
if (log.isDebugEnabled())
    log.debug("Calling startup class " + method);
method.invoke(catalinaDaemon, param);
```

Same here, we call start of catalina object.
```java
    /**
     * Start the Catalina daemon.
     * @throws Exception Fatal start error
     */
    public void start()
        throws Exception {
        if( catalinaDaemon==null ) init();

        Method method = catalinaDaemon.getClass().getMethod("start", (Class [] )null);
        method.invoke(catalinaDaemon, (Object [])null);

    }
```

#### Catalina
Bootstrap main method call Catalina load and start method. The following is the main logic of load method.
As we can see, it does two things: create Digester object and invoke init method of Server.

Catalina.load():
```java
Digester digester = createStartDigester();
inputSource.setByteStream(inputStream);
digester.push(this);
digester.parse(inputSource);
getServer().setCatalina(this);
getServer().init();
```

Catalina.start(): It mainly starts the StandardServer
```java
getServer().start();
```

#### init/start method
Now, we know Catalina class init and start the server, but we don't find init/start method in StandardServer class, thus we know these methods are implemented in this parent class. We go to its parent's parent's class LifecycleBase and find init/start implementation.

LifecycleBase.init()
```java
    @Override
    public final synchronized void init() throws LifecycleException {
        if (!state.equals(LifecycleState.NEW)) {
            invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
        }

        try {
            setStateInternal(LifecycleState.INITIALIZING, null, false);
            initInternal();
            setStateInternal(LifecycleState.INITIALIZED, null, false);
        } catch (Throwable t) {
            handleSubClassException(t, "lifecycleBase.initFail", toString());
        }
    }
```

StandardServer、StandardService、StandardEngine、StandardHost、StandardContext. They all implement abstract class LifecycleBase initInternal(). Similarily, there is startInternal() in start() method.

Let's take a look at StandardServer class initInternal() method. Main logic is as follow:
```java
for (int i = 0; i < services.length; i++) {
    services[i].init();
}
```
startInternal()
```java
// Start our defined Services
synchronized (servicesLock) {
    for (int i = 0; i < services.length; i++) {
        services[i].start();
    }
}
```

Now, let's take a look at StandardService.initInternal() and startInternal()
```java
 @Override
    protected void initInternal() throws LifecycleException {

        super.initInternal();

        if (engine != null) {
            engine.init();
        }

        // Initialize any Executors
        for (Executor executor : findExecutors()) {
            if (executor instanceof JmxEnabled) {
                ((JmxEnabled) executor).setDomain(getDomain());
            }
            executor.init();
        }

        // Initialize mapper listener
        mapperListener.init();

        // Initialize our defined Connectors
        synchronized (connectorsLock) {
            for (Connector connector : connectors) {
                connector.init();
            }
        }
    }
```

```java
    @Override
    protected void startInternal() throws LifecycleException {

        if(log.isInfoEnabled())
            log.info(sm.getString("standardService.start.name", this.name));
        setState(LifecycleState.STARTING);

        // Start our defined Container first
        if (engine != null) {
            synchronized (engine) {
                engine.start();
            }
        }

        synchronized (executors) {
            for (Executor executor: executors) {
                executor.start();
            }
        }

        mapperListener.start();

        // Start our defined Connectors second
        synchronized (connectorsLock) {
            for (Connector connector: connectors) {
                // If it has already failed, don't try and start it
                if (connector.getState() != LifecycleState.FAILED) {
                    connector.start();
                }
            }
        }
    }
```

StandardEngine.startInternal(). start multithreaded to start child
```java
// Start our child containers, if any
Container children[] = findChildren();
List<Future<Void>> results = new ArrayList<>();
for (int i = 0; i < children.length; i++) {
    results.add(startStopExecutor.submit(new StartChild(children[i])));
}
```




-> Connector.initInternal()
```java
protected Adapter adapter = null;
adapter = new CoyoteAdapter(this);
protocolHandler.setAdapter(adapter);
protocolHandler.init();
```
-> AbstractProtocol.init()
```java
private final AbstractEndpoint<S,?> endpoint;
endpoint.init();
```
-> AbstractEndpoint.init()
```java
if (bindOnInit) {
    bindWithCleanup();
    bindState = BindState.BOUND_ON_INIT;
}
```
-> NioEndpoint.bind()  (template pattern)


#### Lifecycle

Lifecycle interface defines init and start methods. Also, it defines the following methods:
```java
    /**
     * Add a LifecycleEvent listener to this component.
     *
     * @param listener The listener to add
     */
    public void addLifecycleListener(LifecycleListener listener);


    /**
     * Get the life cycle listeners associated with this life cycle.
     *
     * @return An array containing the life cycle listeners associated with this
     *         life cycle. If this component has no listeners registered, a
     *         zero-length array is returned.
     */
    public LifecycleListener[] findLifecycleListeners();


    /**
     * Remove a LifecycleEvent listener from this component.
     *
     * @param listener The listener to remove
     */
    public void removeLifecycleListener(LifecycleListener listener);
```
LifecycleListener

```java
public interface LifecycleListener {
    /**
     * Acknowledge the occurrence of the specified event.
     *
     * @param event LifecycleEvent that has occurred
     */
    public void lifecycleEvent(LifecycleEvent event);
}
```
Observer pattern. When any events happen, listeners will get notified. Now, let's take a look at LifecycleEvent. It encapsulates Lifecycle as event source.

```java
public final class LifecycleEvent extends EventObject {

    private static final long serialVersionUID = 1L;


    /**
     * Construct a new LifecycleEvent with the specified parameters.
     *
     * @param lifecycle Component on which this event occurred
     * @param type Event type (required)
     * @param data Event data (if any)
     */
    public LifecycleEvent(Lifecycle lifecycle, String type, Object data) {
        super(lifecycle);
        this.type = type;
        this.data = data;
    }


    /**
     * The event data associated with this event.
     */
    private final Object data;


    /**
     * The event type this instance represents.
     */
    private final String type;


    /**
     * @return the event data of this event.
     */
    public Object getData() {
        return data;
    }


    /**
     * @return the Lifecycle on which this event occurred.
     */
    public Lifecycle getLifecycle() {
        return (Lifecycle) getSource();
    }


    /**
     * @return the event type of this event.
     */
    public String getType() {
        return this.type;
    }
}
```

Remember that in startInteral method. fireLifecycelEvent is a method from LifecycleBase. setState() method basically notifies its listeners.

```java
 @Override
    protected void startInternal() throws LifecycleException {

        fireLifecycleEvent(CONFIGURE_START_EVENT, null);
        setState(LifecycleState.STARTING);
       //...
    }
```

```java
    /**
     * Allow sub classes to fire {@link Lifecycle} events.
     *
     * @param type  Event type
     * @param data  Data associated with event.
     */
    protected void fireLifecycleEvent(String type, Object data) {
        LifecycleEvent event = new LifecycleEvent(this, type, data);
        for (LifecycleListener listener : lifecycleListeners) {
            listener.lifecycleEvent(event);
        }
    }


    @Override
    public void addLifecycleListener(LifecycleListener listener) {
        lifecycleListeners.add(listener);
    }
```

#### StandardServer.await() and stopServer()
Tomcat has one user thread and five daemon threads. Once the user thread stops, the whole appliation stops.

In StandardServer.await(), it actaully listens on port 8005 and waiting for SHUTDOWN command. 

Catalina has a method called stopServer. It send SHUTDOWN to the socket.


### Request flow

#### Convert socket to internal request object
In Tomcat Catalina class, it will trigger ConnectorCreateRule class begin() method.
```java
digester.addRule("Server/Service/Connector",
                 new ConnectorCreateRule());
digester.addRule("Server/Service/Connector",
                 new SetAllPropertiesRule(new String[]{"executor"}));
digester.addSetNext("Server/Service/Connector",
                    "addConnector",
                    "org.apache.catalina.connector.Connector");
```

begind() method:
```java
 // ...
 Connector con = new Connector(attributes.getValue("protocol"));
 // ...
```
By default, attribute protocol is read from server.xml
```xml
<Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />

<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
```

Connector constructor will initialize a protocolHandler instance. ProtocolHandler will initailize an Endpoint

In Connector class startInternal method, it calls protocolHandler start() method. Http11Protocol doesn't have start method, so it will invoke its parent class AbstractProtocol start() method.
```java
    @Override
    public void start() throws Exception {
        if (getLog().isInfoEnabled()) {
            getLog().info(sm.getString("abstractProtocolHandler.start", getName()));
            logPortOffset();
        }

        endpoint.start();
        monitorFuture = getUtilityExecutor().scheduleWithFixedDelay(
                new Runnable() {
                    @Override
                    public void run() {
                        if (!isPaused()) {
                            startAsyncTimeout();
                        }
                    }
                }, 0, 60, TimeUnit.SECONDS);
    }
```
As we see above, AbstractProtocol starts Endpoint (e.g. NioEndpoint)

```java
    /**
     * Start the NIO endpoint, creating acceptor, poller threads.
     */
    @Override
    public void startInternal() throws Exception {

        if (!running) {
            running = true;
            paused = false;

            processorCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                    socketProperties.getProcessorCache());
            eventCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                            socketProperties.getEventCache());
            nioChannels = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                    socketProperties.getBufferPool());

            // Create worker collection
            if ( getExecutor() == null ) {
                createExecutor();
            }

            initializeConnectionLatch();

            // Start poller threads
            pollers = new Poller[getPollerThreadCount()];
            for (int i=0; i<pollers.length; i++) {
                pollers[i] = new Poller();
                Thread pollerThread = new Thread(pollers[i], getName() + "-ClientPoller-"+i);
                pollerThread.setPriority(threadPriority);
                pollerThread.setDaemon(true);
                pollerThread.start();
            }

            startAcceptorThreads();
        }
    }
```
Endpoint starts multiple pollers thread and multiple acceptor thread. Acceptor spawns ServerSocket. And register the newly created socket to poller. It converts socket to a socketWrapper.

```java
 /**
         * Registers a newly created socket with the poller.
         *
         * @param socket    The newly created socket
         */
        public void register(final NioChannel socket) {
            socket.setPoller(this);
            NioSocketWrapper socketWrapper = new NioSocketWrapper(socket, NioEndpoint.this);
            socket.setSocketWrapper(socketWrapper);
            socketWrapper.setPoller(this);
            socketWrapper.setReadTimeout(getConnectionTimeout());
            socketWrapper.setWriteTimeout(getConnectionTimeout());
            socketWrapper.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
            socketWrapper.setSecure(isSSLEnabled());
            PollerEvent r = eventCache.pop();
            socketWrapper.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
            if (r == null) {
                r = new PollerEvent(socket, socketWrapper, OP_REGISTER);
            } else {
                r.reset(socket, socketWrapper, OP_REGISTER);
            }
            addEvent(r);
        }
```

Now, let's take a look at Http11Processor.service method and Http11InputBuffer.parseRequestLine() method. These two methods bacially construct a org.apache.coyote.Request object with the socket.


#### Request object from connector to Engine, Host, Context, Wrapper

In Http11Processor.service method, we have the following code:
```java
// Process the request in the adapter
            if (getErrorState().isIoAllowed()) {
                try {
                    rp.setStage(org.apache.coyote.Constants.STAGE_SERVICE);
                    getAdapter().service(request, response);
//...
```
AbstractProtocol create processor and it passes its adapter attribute to processor. See below: 
```java
    @Override
    protected Processor createProcessor() {
        Http11Processor processor = new Http11Processor(this, adapter);
        return processor;
    }
```
So, where does adapter is created for abstract protocol? It's in Connector initInternal method.

```java
 // Initialize adapter
adapter = new CoyoteAdapter(this);
protocolHandler.setAdapter(adapter);
```

Now, we knows that processor service method actually calls service method in org.apache.catalina.connector.CoyoteAdapter.
```java
public void service(org.apache.coyote.Request req, org.apache.coyote.Response res) {
    //...
    // Parse and set Catalina and configuration specific
    // request parameters
    postParseSuccess = postParseRequest(req, request, res, response);
}
```
This service method convert org.apache.coyote.Request to org.apache.catalina.connector.Request. The postParseRequest method set attributes of connector.Request object. 

Inisde CoyoteAdapter.postParseRequest method, it calls connector's mapper class method map.
```java
// This will map the the latest version by default
connector.getMapper().map(serverName, decodedURI, version, request.getMappingData());
```

And map method calls internalMap method. what internalMap does is to set mappingData such as mappingData.host、mappingData.contextPath、mappingData.contexts、mappingData.wrapper

Next, we'll take a look at how host, context, wrapper get loaded. As we mentioned earlier, StandardService.startInternal method calls mapperListener.start(). Let's take a look at startInternal method of mapper listener.

```java
 @Override
    public void startInternal() throws LifecycleException {

        setState(LifecycleState.STARTING);

        Engine engine = service.getContainer();
        if (engine == null) {
            return;
        }

        findDefaultHost();

        addListeners(engine);

        Container[] conHosts = engine.findChildren();
        for (Container conHost : conHosts) {
            Host host = (Host) conHost;
            if (!LifecycleState.NEW.equals(host.getState())) {
                // Registering the host will register the context and wrappers
                registerHost(host);
            }
        }
    }
```
Tomcat regsiters Host, Context, Wrapper into a Mapper object. One Service has one Engine and multiple connectors. One Engine has mutliple Host. One host has multiple Context. One Context has multiple Wrapper. Tomcat's Map mechanism maps the incoming request to the correct Wrapper.


### Pipeline and Valve

Tomcat uses Chain of Responsibility Pattern to improve compoennt flexiblility and scalability. 

Tomcat defines a Pipeline and a Valve. Pipeline is used to construct chain of responsibility and valve represents processor on its chain.

{% asset_img pipeline.png pipeline %}


After Tomcat's Connector converts Request, it will handle the request to Engine's Pipeline and Valve. Request in Engine's Pipeline will be delivered to Host Pipeline, etc.

Inside service method of CoyoteAdapter, there is the following invocation.
```java
 connector.getService().getContainer().getPipeline().isAsyncSupported());
```
There is an addValve method in StandardPipeline class. In the default constructor of StandardEngine:
```java
/**
     * Create a new StandardEngine component with the default basic Valve.
     */
    public StandardEngine() {

        super();
        pipeline.setBasic(new StandardEngineValve());
        /* Set the jmvRoute using the system property jvmRoute */
        try {
            setJvmRoute(System.getProperty("jvmRoute"));
        } catch(Exception ex) {
            log.warn(sm.getString("standardEngine.jvmRouteFail"));
        }
        // By default, the engine will hold the reloading thread
        backgroundProcessorDelay = 10;

    }
```

StandardEngineValve's invoke method calls StandardHost's pipeline, and StandardContext, StandardWrapper. 
```java
 @Override
    public final void invoke(Request request, Response response)
        throws IOException, ServletException {

        // Select the Host to be used for this Request
        Host host = request.getHost();
        if (host == null) {
            // HTTP 0.9 or HTTP 1.0 request without a host when no default host
            // is defined. This is handled by the CoyoteAdapter.
            return;
        }
        if (request.isAsyncSupported()) {
            request.setAsyncSupported(host.getPipeline().isAsyncSupported());
        }

        // Ask this Host to process this request
        host.getPipeline().getFirst().invoke(request, response);
    }
```

### Web deployment

There is no context configured in server.xml. How does tomcat deploy context? We all knows that Tomcat server find the context object (default: StandardContext) based on url. In Tomcat, all default container component StandardEngine, StandardHost, StandardContext, StandardWrapper inherit org.apache.catalina.core.ContainerBase. These default container component all call parent's startInternal method in their own startInternal method.

startInternal method in ContainerBase:
```java
  // Start the Valves in our pipeline (including the basic), if any
        if (pipeline instanceof Lifecycle) {
            ((Lifecycle) pipeline).start();
        }

        setState(LifecycleState.STARTING);

        // Start our thread
        if (backgroundProcessorDelay > 0) {
            monitorFuture = Container.getService(ContainerBase.this).getServer()
                    .getUtilityExecutor().scheduleWithFixedDelay(
                            new ContainerBackgroundProcessorMonitor(), 0, 60, TimeUnit.SECONDS);

```

We navigate to ContainerBackgroundProcessor class. 
```java
protected void processChildren(Container container) {
            ClassLoader originalClassLoader = null;

            try {
                if (container instanceof Context) {
                    Loader loader = ((Context) container).getLoader();
                    // Loader will be null for FailedContext instances
                    if (loader == null) {
                        return;
                    }

                    // Ensure background processing for Contexts and Wrappers
                    // is performed under the web app's class loader
                    originalClassLoader = ((Context) container).bind(false, null);
                }
                container.backgroundProcess();
                Container[] children = container.findChildren();
                for (int i = 0; i < children.length; i++) {
                    if (children[i].getBackgroundProcessorDelay() <= 0) {
                        processChildren(children[i]);
                    }
                }
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                log.error(sm.getString("containerBase.backgroundProcess.error"), t);
            } finally {
                if (container instanceof Context) {
                    ((Context) container).unbind(false, originalClassLoader);
                }
            }
        }
```
This method call its backgroundProcess method and then process its children. The backgroundProcess in ContainerBase will fire an PERIODIC_EVENT event.

In Catalina class, there is one line of code:
```java
digester.addRuleSet(new HostRuleSet("Server/Service/Engine/"));
```

org.apache.catalina.startup.HostConfig is added as a listern to StandardHost.

In lifecycleEvent method
```java
 // Process the event that has occurred
        if (event.getType().equals(Lifecycle.PERIODIC_EVENT)) {
            check();
        } else if (event.getType().equals(Lifecycle.BEFORE_START_EVENT)) {
            beforeStart();
        } else if (event.getType().equals(Lifecycle.START_EVENT)) {
            start();
        } else if (event.getType().equals(Lifecycle.STOP_EVENT)) {
            stop();
        }
```
The check method will deploy web app.

```java
  /**
     * Deploy applications for any directories or WAR files that are found
     * in our "application root" directory.
     */
    protected void deployApps() {

        File appBase = host.getAppBaseFile();
        File configBase = host.getConfigBaseFile();
        String[] filteredAppPaths = filterAppPaths(appBase.list());
        // Deploy XML descriptors from configBase
        deployDescriptors(configBase, configBase.list());
        // Deploy WARs
        deployWARs(appBase, filteredAppPaths);
        // Deploy expanded folders
        deployDirectories(appBase, filteredAppPaths);

    }
```

Navigate into deploy methods
```java
    Class<?> clazz = Class.forName(host.getConfigClass());
    LifecycleListener listener = (LifecycleListener) clazz.getConstructor().newInstance();
    context.addLifecycleListener(listener);

    context.setName(cn.getName());
    context.setPath(cn.getPath());
    context.setWebappVersion(cn.getVersion());
    context.setDocBase(cn.getBaseName() + ".war");
    host.addChild(context);
```

host.getConfigClass returns a ContextConfig object. The addChild method will start the parameter Context object. The starting process will emit a set of events: BEFORE_INIT_EVENT、AFTER_INIT_EVENT、BEFORE_START_EVENT、CONFIGURE_START_EVENT、START_EVENT、AFTER_START_EVENT.

ContextConfig was added as a listener into context. So ContextConfig will be triggered.

```java
// Process the event that has occurred
        if (event.getType().equals(Lifecycle.CONFIGURE_START_EVENT)) {
            configureStart();
        } else if (event.getType().equals(Lifecycle.BEFORE_START_EVENT)) {
            beforeStart();
        } else if (event.getType().equals(Lifecycle.AFTER_START_EVENT)) {
            // Restore docBase for management tools
            if (originalDocBase != null) {
                context.setDocBase(originalDocBase);
            }
        } else if (event.getType().equals(Lifecycle.CONFIGURE_STOP_EVENT)) {
            configureStop();
        } else if (event.getType().equals(Lifecycle.AFTER_INIT_EVENT)) {
            init();
        } else if (event.getType().equals(Lifecycle.AFTER_DESTROY_EVENT)) {
            destroy();
        }
```
The configureStart method will parse web.xml to a webXml object to get servlet and filter configuration. And it will set context object.


When will Servlet, Listener, Filter get started? It's in StandardContext.startInternal().

The startInternal() of StandardContext
```java
 // Notify our interested LifecycleListeners
fireLifecycleEvent(Lifecycle.CONFIGURE_START_EVENT, null);
//...
// Configure and call application event listeners
            if (ok) {
                if (!listenerStart()) {
                    log.error(sm.getString("standardContext.listenerFail"));
                    ok = false;
                }
            }

// Configure and call application filters
            if (ok) {
                if (!filterStart()) {
                    log.error(sm.getString("standardContext.filterFail"));
                    ok = false;
                }
            }

            // Load and initialize all "load on startup" servlets
            if (ok) {
                if (!loadOnStartup(findChildren())){
                    log.error(sm.getString("standardContext.servletFail"));
                    ok = false;
                }
            }
```


A request will call org.apache.catalina.core.StandardWrapperValve invoke method eventually. It'll call Servlet service method.


### Connector design

The responsibilities of Connector includes:
* Listen requests
* Parse requests with correct protocol
* Map requests to the correct container
* response to client

Tomcat supports HTTP and AJP. It supports mutliple I/O as well, include BIO, NIO, APR, NIO2 and HTTP/2 protocol. 

{% asset_img connector.png connector %}

Http11NioProtocol represent NIO and http protocol handler. Endpoint represents different I/O. Processor represents different protocols.

Endpoint accepts request and asks Processor to process the requests. Processor then map the request to the correct container.

Tomcat uses Mapper and MapperListener tow classes. 

### Executor

Executor is used to maintain a thread pool.

### Bootstrap and Catalina

Catalina class provides a shell program. This program parse the server.xml file and start, stop application server.

Bootstrap is the entrance to the application server. It creates Catalina instances.

{% asset_img start.png start %}

## Class loader

### J2SE class loader

We all knows that JVM provides three class laoders. Bootstrap class loader -> Extension class loader -> System class loader.

* Bootstrap: load JAVA_HOME/jre/lib
* Extension: load JAVA_HOME/jre/lib/ext
* System: load CLASSPATH or -classpath

### Tomcat class loader

Servlet standard requires that each web application has a independent class laoder instance.
* Isolation: If different web apps use different class loader, it avoids interference between dependencies.
* Flexibility: We can restart one web app without affecting another web app.
* Performance: One web app doesn't need to scan other dependencies.

{% asset_img class-loader.png class loader %}

* Common: path common.loader -> CATALINA_HOME/lib. 
```
ClassLoader commonLoader = null;
commonLoader = createClassLoader("common", null);
```
* Catalina: path server.loader
* Shared: path shared.loader
* Web app: load /WEB-INF/classes and /WEB-INF/lib


# Catalina

## What is Catalina

Catalina includes all components mentioned earlier. It is the core of Tomcat. 

## Digester

Catalina uses Digester to parse XML (server.xml)

## Create Server

```java
<?xml version='1.0' encoding='utf-8'?>
<Server port="8005" shutdown="SHUTDOWN">

  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <Listener className="org.apache.catalina.core.JasperListener" />
  <Listener className="org.apache.catalina.mbeans.ServerLifecycleListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />

  <Listener className="com.springsource.server.web.tomcat.ServerLifecycleLoggingListener"/>

  <Service name="Catalina">
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
	       redirectPort="8443" />

    <Connector port="8443" protocol="HTTP/1.1" SSLEnabled="true"
	       maxThreads="150" scheme="https" secure="true"
	       clientAuth="false" sslProtocol="TLS"
	       keystoreFile="config/keystore"
	       keystorePass="changeit"/>

    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />

    <Engine name="Catalina" defaultHost="localhost">

      <Realm className="org.apache.catalina.realm.JAASRealm" appName="dm-kernel"
	     userClassNames="com.springsource.kernel.authentication.User"
	     roleClassNames="com.springsource.kernel.authentication.Role"/>

      <Host name="localhost"  appBase="webapps"
	    unpackWARs="true" autoDeploy="true"
	    xmlValidation="false" xmlNamespaceAware="false">

        <Valve className="org.apache.catalina.valves.AccessLogValve" 
               directory="serviceability/logs/access"
	       prefix="localhost_access_log." suffix=".txt" pattern="common" 
               resolveHosts="false"/>
					
        <Valve className="com.springsource.server.web.tomcat.ApplicationNameTrackingValve"/>
      </Host>
    </Engine>
  </Service>
</Server>
```

{% asset_img catalina.png catalina %}

A Engine is the highest-level of a container. It can contains one or more Hosts. You could configure a Tomcat server to run on several hostnames, known as virtual host. The Catalina Engine receives HTTP requests from the HTTP connector, and direct them to the correct host based on the hostname/IP address in the request header.

A Realm is a database of user, password, and role for authentication (i.e., access control). You can define Realm for any container, such as Engine, Host, and Context, and Cluster. It uses the JNDI name UserDatabase defined in the GlobalNamingResources. Besides the UserDatabaseRealm, there are: JDBCRealm (for authenticating users to connect to a relational database via the JDBC driver); DataSourceRealm (to connect to a DataSource via JNDI; JNDIRealm (to connect to an LDAP directory); and MemoryRealm (to load an XML file in memory).

A Host defines a virtual host under the Engine, which can in turn support many Contexts (webapps).

Tomcat supports server clustering. It can replicate sessions and context attributes across the clustered server. It can also deploy a WAR-file on all the cluster.

A Valve can intercept HTTP requests before forwarding them to the applications, for pre-processing the requests. A Valve can be defined for any container, such as Engine, Host, and Context, and Cluster. In the default configuration, the AccessLogValve intercepts an HTTP request and creates a log entry in the log file.

## Deploy Web

Catalina mainly uses StandardHost, HostConfig, StandardContext, ContextConfig, StandardWrapper to deploy webapp.

## Hendle Web Requests

As we mentioned, Tomcat use org.apache.tomcat.util.http.mapper.Mapper matain the association between Connector and Container (Host, Context, Wrapper). And org.apache.catalina.connector.MapperListener listens on events for container and dynamically register or remove corresponding association.

When connector receives a request, it reads request and invoke CoyoteAdapter.service() method.


# Coyote

Coyote is the name of Tomcat Connector framework. It encapsulates underlying network communication(Socket). Coyote converts Socket input into Request object and gives it to Catalina container and converts response into socket output.

Coyote is an independent module. It has no relationship with Servlet standard. Thus, its Request and Response objects don't implement Servlet standard.

Coyote supports the following three protocols in application layer:
* HTTP/1.1
* AJP
* HTTP/2.0

Also, it supports the following I/O in terms of transport layer.
* NIO
* NIO2
* APR

## Key concepts

* Endpoint

Encapsulates Socket. Tomcat provides AbstractEndpoint and provide NioEndpoint, AprEndpoint and Nio2Endpoint three implementation.

* Processor

Construct Request and Response object and pass them to Catalina container through Adapter. Http11Processor, AjpProcessor, Stream Preocessor.

* ProtocolHandler

Encapsulate Endpoint and Processor. Http11NioProtocol, Http11Nio2Protocol, etc.

* UpgradeProtocol

## Simplified flow

Endpoint -> Endpoint.Acceptor -> SocketProcessorBase -> Endpoint.Handler process(SocketWrapper) -> Processor -> Adapter service(request, response) -> Valve

Connector starts Endpoint instances. Endpoint run multiple threads and each thread runs an AbstractEndpoint.Acceptor instance. Acceptr converts Socket to SocketWrapper instance and handles it to SocketProcessor. 


# Cluster

{% asset_img cluster.png Cluster %}


First, org.apache.catalina.Cluster is responsible for setting up a way to communicate within the Cluster.
