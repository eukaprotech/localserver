# Description
An android server based on ServerSocket.

# Versions
* [V1.0.0](https://github.com/eukaprotech/localserver/blob/master/com/eukaprotech/localserver/localserver/1.0.0/README.md "Version 1.0.0 Overview")

# Getting Started (V1.0.0)
Add the dependency in build.gradle (App module)

```compile 'com.eukaprotech.localserver:localserver:1.0.0@aar'```

Add permission in manifest file

```<uses-permission android:name="android.permission.INTERNET" />```

# Usage (V1.0.0)

```javascript
    localServer = new LocalServer(context) {
        @Override
        public int onInitPort() {
            return 8888;  //Set preferred port.  Port 0 means to select an arbitrary unused port
        }

        @Override
        public void onConfigured(String ip_address, int port) {
            // ip_address (host) and port, the server is running on. You can display these values for purpose of browsing the server from a browser or a http client
        }

        @Override
        public void onError(Exception e) {
             //log errors from Exceptions thrown
        }

        @Override
        public boolean appDebug() {
            return false;  //true implies the server will respond with detailed debug errors; for every 500 Internal Server Error response 
        }
    };
    
    localServer.start(); // also used to restart the server when stoped
 ```
   
Remember to shut down the server:

```javascript
    localServer.stop();
```
    
 To check whether server is running:
 
 ```javascript
    localServer.isRunning();
 ```
    
# Routing

```javascript
    localServer.route("path", HttpHandler);  //first input is the path, second is a class implementing a HttpHandler interface

    localServer.route("/", new HttpHandler() {    // example using an anonymous class implementing HttpHandler interface
        @Override
        public Response onHandle(Exchange exchange) {
            return ResponseHandler.string("content");
        }
    });
```
    
The HttpHandler interface declares only one function onHandle from which you get the object of class Exchange. The object, Exchange, contains information from a client request such as:

* path (the path set on routing e.g "/home")
* URI  (the complete browsable link from the above path e.g "http://192.168.0.34:8888/home")
* port (the port the server is running on e.g 8888)
* ip_address (the ip_address the server is running on e.g "192.168.0.34")
* client ip_address
* request method (GET, POST,)
* request Headers (from the client)
* parameters (from the client; both query parameters and post parameters bundled together)
* route parameters (captured segments of the URI determined by the routing syntax used)
* response Headers (to be sent to the client)

The onHandle function requires a return of an object of a class implementing Response interface. This response will be sent to the client. To build the response easily, a class ResponseHandler is used. Sample responses include:

```javascript
* ResponseHandler.string("content");  // a String response. Can be a html string  
* ResponseHandler.asset("file name");  //To return a file saved in android asset folder
* ResponseHandler.file(new File("")); //To return a file
* ResponseHandler.JSON(new JSONObject()); ResponseHandler.JSON(new JSONArray());  //To return JSON object or array
* ResponseHandler.redirect("path");  // to redirect to a given path with a default 302 temporary-redirect
* ResponseHandler.redirect("path", 301); //a 301 permanent-redirect
* ResponseHandler.redirect("path", new HashMap<String, Object>(){}); // a redirect with the given query parameters
* ResponseHandler.redirect("path", 301, new HashMap<String, Object>(){}); // a 301 redirect with the given query parameters
* ResponseHandler.redirectBack(); // redirect back
* ResponseHandler.redirectBack(new HashMap<String, Object>(){}); // redirect back with the given query parameters
* ResponseHandler.methodNotAllowed(new ArrayList<REQUEST_METHOD>(){}); //405 Method Not Allowed response with a list of allowed methods
* ResponseHandler.requireAuthBasic("realm"); // a 401 Access Denied response
* ResponseHandler.throw404(); // a 404 Not Found response
* null; // equivalent to ResponseHandler.throw404();
```

# Route Parameters

To capture segments of the URI within your route:

```javascript
    localServer.route("/client/{client_id}", new HttpHandler() {
        @Override
        public Response onHandle(Exchange exchange) {
            //To capture the client_id, get the route parameters from the exchange.
            HashMap<String, Object> route_params = exchange.getRouteParameters();
            
            Object client_id = route_params.get("client_id"); //OR 
            String client_id = String.valueOf(route_params.get("client_id"));
            
            
            return ResponseHandler.string("content");
        }
    });
```
    
To set Regular Expression Constraints on the parameter add the splitter '|' followed by the constraint:

```javascript
    localServer.route("/client/{client_id|[0-9]+}", new HttpHandler() {  // ensures that client_id must be numeric
        @Override
        public Response onHandle(Exchange exchange) {
            //To capture the client_id, get the route parameters from the exchange.
            HashMap<String, Object> route_params = exchange.getRouteParameters();
            
            Object client_id = route_params.get("client_id"); //OR 
            String client_id = String.valueOf(route_params.get("client_id"));
            
            
            return ResponseHandler.string("content");
        }
    });
```
    
You may define as many route parameters as required by your route:

```javascript
    localServer.route("/client/{client_id}/comments/{comment}", new HttpHandler() {
        @Override
        public Response onHandle(Exchange exchange) {
            
        }
    });
```

# Routing the 404 Not Found

To create a custom response for a '404 Not Found':

```javascript
    localServer.route404(new HttpHandler() {
        @Override
        public Response onHandle(Exchange exchange) {
            return ResponseHandler.string("content"); // This will be the response every time you throw a 404 error response using ResponseHandler.throw404(); or when the server does not find a set routing for a request from a client.
        }
    });
```

# Creating a Catch-All route (Intercepting the 404 Not Found)       

A catch-all route can be used to catch all client requests that are not routed. This intercepts the 404 not found route.

```javascript
    localServer.route404Intercept(new HttpHandler() {
        @Override
        public Response onHandle(Exchange exchange) {
            //Here you can filter URIs that are crucial and return their respective response types.
            //You can return a ResponseHandler.throw404(); for URIs that deserve a '404 Not Found' response.
            return ResponseHandler.string("content");
        }
    });
```
    
# Capture Basic Auth Credentials 

The basic auth value from a client can be catured from the request headers of the Exchange. To easen this process a RequestHandler class is used:

```javascript
    HashMap<String, String> credentials = RequestHandler.getBasicAuthCredentials(exchange);
    if(credentials != null){
        String username = credentials.get("username");
        String password = credentials.get("password");
    }
```
    
# Capture Files from Parameters

Let us assume that a .png image file has been posted among the parameters by a client using the key "avatar":

```javascript
    HashMap<String, Object> parameters = exchange.getParameters();
    Object avatar = parameters.get("avatar"); 
    if (avatar instanceof HashMap) { // A value of type HashMap signifies a file from parameters
        HashMap<String, Object> fileMap = (HashMap<String, Object>) avatar;
        String filename = fileMap.get("filename").toString();
        String contentType = fileMap.get("Content-Type").toString();
        Object _file = fileMap.get("file");
        ByteArrayOutputStream byteArrayOutputStream = (ByteArrayOutputStream) _file;
        File documentsDir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOCUMENTS);
        File file = new File(documentsDir, "the_avatar_image.png");
        try {
            FileOutputStream outF = new FileOutputStream(file);
            byteArrayOutputStream.writeTo(outF);
            try {
                outF.close();
            } catch (Exception ex) {
            }
        } catch (Exception ex) {}
    }
```
    
# Note

* Considering this is a local server, only devices under the same local network as the device hosting the server can browse the server.
* In case of conflict between a route with route-parameters and one without, the server gives priority to the route without.
* The server does not allow slashes in route parameters regardless of how you define the constraint. In case you require parameters with slashes, utilise query parameters.
* If both the server and client are on the same device, avoid letting the server overwrite the files sent by the client; definitely this will corrupt the transfer of the files. 
* The server can handle multiple clients through multithreading. 
* One of the applications of this server would be creating a local media server; whereby you can share/browse media files from a device. Any frequently required resources such as javascript or css files can be bundled in an android app under the assets folder and returned to clients using ResponseHandler.asset("file name");.

[ ![Download](https://api.bintray.com/packages/eukaprotech/maven/localserver/images/download.svg) ](https://bintray.com/eukaprotech/maven/localserver/_latestVersion)
