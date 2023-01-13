# Lua HTTP Request Examples

This project will show how to use the sendHttp build-in function to query sensor data.  

## Table of Contents

* [Lua HTTP Request Examples](#lua-http-request-examples)
     *  [Table of Contents](#table-of-contents)
     *  [Overview](#overview)   
     *  [Formatting a Request](#formatting-a-request)
     *  [Simple Example](#simple-example)   
     *  [Bearer Token example](#bearer-token-example)
     *  [SOAP Example](#soap-example)
     *  [Summary](#summary)
     *  [Contributing](#contributing)
     *  [License](#license)
     *  [Additional Resources](#additional-resources)   
      

## Overview 

![image-20220822152746080](Images/image-20220822152746080.png)

HTTP is a protocol that can be used to fetch data from a Web server.   This data might be sensor or weather data or any type other type of data really.  This project will show a couple examples of how to issue HTTP requests from a Lua window in ESP and retrieve the results to create events.  An HTTP request can be broken down into these core components: 

+ **url** - Specifies the URL of the request 
+ **method** - HTTP method to use (`GET`, `PUT`, `POST`, `DELETE`)
+ **headers** - Name value pairs that are passed to the server.  (`"Content-Type"` = `"application/json"`)
+ **body** -  Data to be passed to the server.  Only valid for `PUT` and `POST`. 

This data is generally returned as 

- a binary buffer
- plain text 
- JSON

These formats are specified in the headers table which is part of the request that is sent to the server.   The process of sending a request starts with formatting a Lua table which contains all the information required.   Please review the [SAS Event Stream Processing Documentation](https://go.documentation.sas.com/doc/en/espcdc/v_027/espcreatewindows/p0yj92wgv3ssyyn1syatsh9l1t74.htm#n0ppam7yzjg4b3n1jpn3igkaywt5) for complete formatting details. 

## Formatting a Request

Before we dive into full examples, let's take a look at how to format a POST request which will return an authorization token.   As stated earlier, the input for the sendHttp function is a Lua table which contains the core components listed above.   Some servers require a 2 step authorization.  The first step is a POST request to acquire a token which is good for a set period of time.  The returned token is then used to issue GET requests to acquire data.   If you wrote a Lua function for this purpose it might look like this: 

```lua
function getToken(url_auth)
      local   request = {}  -- main requests table 
      local   headers = {}  -- headers table 

      headers["Content-Type"] = "application/json"
      request["headers"] = headers
      request["url"] = url_auth  -- url for request 
      request["method"] = "POST"
      request["body"] = "{\"username\": \"your.name@sas.com\", \"password\": \"Orion123\"}"
      request["tolua"] = true
      local   response = sendHttp(request)
      return response.response.token
 end  -- function
```



The **sendHttp** function returns JSON or a Lua table containing the status of the request and the response.  A status of 200 is good, everything else is bad.  For this request a token is returned looks similar to the following.  Remember, these tokens are only valid for a set period of time.  

```
Start of auth token 
eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJ0b20udHVuaW5nQHNhcy5jb20iLCJzY29wZXMiOlsiQ1VTVE9NRVJfVVNFUiJdLCJ1c2VySWQiOiJlYzcwY2YyMC1kMThlLTExZWMtYjM1ZC1hNWIwNjU2NTE3Y2YiLCJlbmFibGVkIjp0cnVlLCJpc1B1YmxpYyI6ZmFsc2UsInRlbmFudElkIjoiYjEyMzNiMTAtYWEyOC0xMWU4LTg2MjQtNzVlOGMyZTI4ZmQ1IiwiY3VzdG9tZXJJZCI6ImJhYjIzMDYwLTgwNDktMTFlYi1iNjg1LWJkNTg3NWJkNDYyNCIsImlzcyI6InRoaW5nc2JvYXJkLmlvIiwiaWF0IjoxNjYyNzQ3ODYwLCJleHAiOjE2NjI3NTY4NjB9.12CkH5UrpEPUhaoZuD9dIqdJrL58a6rDac7pcvF4ehW2ynkuWeeo4udd9x--taddRRnUg4pPuYCHhHvs-C8_9g
End of auth token 
```

So what is missing from the function above?   Since we are calling this function from ESP, what happens when the HTTP server never responds?   What happens when the server responds with something other than a 200?   If the server never responds the ESP project could hang and potentially fail.   If the response is something other than a 200 we might want to invoke some type of retry logic.  A better function might look like this: 

```lua
function getToken(context,url_auth)
      local   request = {}  -- main requests table 
      local   headers = {}  -- headers table 
      local   connect_timeout = 3000
      local   data_timeout = 5000
        
      request["connect-timeout"] = connect_timeout
      request["data-timeout"] = data_timeout

      headers["Content-Type"] = "application/json"
      request["headers"] = headers
      request["url"] = url_auth  -- url for request 
      request["method"] = "POST"
      request["body"] = "{\"username\": \"your.name@sas.com\", \"password\": \"Orion123\"}"
      request["tolua"] = true
      local  response = sendHttp(request)
      local  status = response.status
      if status == 200 then 
          return response.response.token
      else 
        logMessage(context.window,"Auth token retrieval failed with status code  "..tostring(status), "info",debug.getinfo(1).currentline)
        return false
      end     
 end  -- function
```

Now we have solved the hang problem by adding 2 extra parameters to the request:

- **connect-timeout**
- **data-timeout**

This will prevent ESP from hanging endlessly possibly causing a server failure.   The function also includes logic to return false when a 200 status code is not returned.  This allows the calling program a chance to invoke retry logic if desired.   Lastly, let's write a message to the ESP server log just in case diagnostics are required.  For more information on [Passing Lua Log Messages to the ESP Server](https://go.documentation.sas.com/doc/en/espcdc/v_027/espcreatewindows/p0yj92wgv3ssyyn1syatsh9l1t74.htm#p1jx8wo8si4xijn1p8lr88kd080l) reference the Help Center.   

Testing our error logic is easy.  Just change the password to an invalid one.  You will now see a message similar to this in the ESP server log. 

```lua
2022-09-12T15:07:31.252Z
INFORMATIONAL
  Auth token retrieval failed with status code  401 (Lua line 62) 
```

  

## Simple Example

For starters we will examine the simple case where we will issue an HTTP GET to a weather API which returns a JSON string.  We can then parse a few fields from the string and produce a table as seen below:  This example does not require passwords nor bearer tokens.   

![image-20220913155633905](Images/image-20220913155633905.png)Here I am using the heartbeat function instead of the create function.  This provides a sort of built in retry logic which I will explain later.  There are two entry points into your Lua module:

![image-20220913161530586](Images/image-20220913161530586.png)	

```lua
function hb(context)   --  heartbeat function 
                local current = os.time()
                if (current - eventTime >= eventInterval )
                then
                    local events = {}      
                    local   guid = getGuids()[1]  -- base key for events 
                    local   counter = 1
                    local   data = getResults(context,url)
                    
                    if data then -- results are available 
                       print("Generating event")    
                       local e = {}   -- create new event 
                       e.id = guid..":"..tostring(counter)
                       e.deviceid = "weather"
                       e.timestamp = os.time()
                       e.wind_speed = tostring(data.wind.speed)
                       e.temperature = tostring(data.main.temp)
                       e.humidity = tostring(data.main.humidity)
                       e.pressure = tostring(data.main.pressure)
                       events[counter] = e 
                       eventTime = os.time()
                    else   events = nil  -- no response, generate no events 
                    end     

                return (events)    
                end
            end
```

The create function is called when an event flows down into the Lua window as ESP processes incoming events.  The heartbeat function is called each time the project level heartbeat is scheduled to occur.  By default this happens every 1 second and can be set at the project level for each ESP project.  Our weather calls are scheduled based on the  **eventInterval** variable setting.   The **getResults** function either returns data or nil.  In the nil case, an ESP log message is produced.  Only when data is successfully returned do we update **eventTime**.  Therefore, when the REST request fails this code will retry every second while writing log messages defining the reason for the failure until the call succeeds.   

Our REST interface is robust because it contains:

- connection timeout values
- data timeout value 
- failure logging 
- retry logic

A complete ESP project example is located in the ESP_Projects directory under [weather_lua.xml ](./ESP_Projects/weather_lua.xml)



## Bearer Token example

When the server being queried has implemented OAuth 2.0  your HTTP requests will travel over HTTPS and most likely will require a bearer token.  You might have noticed that the previous example used HTTP.  Therefore, our REST call flow will first include a call to an Authentication Server which will generate a bearer token.  This token authorizes the client to communicate with the data server. 

![image-20220930154111272](Images/image-20220930154111272.png)

​	

Using bearer tokens simplifies the API transactions with the data server because they do not need to implement any client authentication or implement cryptographic signing of requests.   The tradeoff is that any client which possesses the token will be considered valid.   To mitigate this tradeoff tokens are short lived and all communications must be secured using HTTPS.  

In this example, we will add the following logic to first acquire the bearer token based on a new variable called **tokenInterval**. 

```lua
if (current - tokenTime >= tokenInterval)
                then
                    token = getToken(context,url_auth)
                    if token then tokenTime = os.time()
                    else tokenTime = 0  -- token not acquired :(  
                    end     
                end

                if (current - eventTime >= eventInterval and token )
                then
                    print("Generating events")
                    local   guid = getGuids()[1]
                    local   counter = 1
```

If the token was acquired successfully, the code will make calls to the data server and parse the results creating a table as follows: 

![image-20220930154147969](Images/image-20220930154147969.png)	



A complete ESP project example is located in the ESP_Projects directory under [Intellisense_lua.xml](./ESP_Projects/Intellisense_lua.xml)



## SOAP Example

In this SOAP example we will make a POST request to a test server which will return the word corresponding to the positive number passed as parameter. Building on what we learnt above we can change the request url as mentioned below.

We can use random function to generate a number between 1 & 100. The method now will be 'POST' and observe that content type is "text/xml; charset=utf-8".

Body is also added to the request and the generated random number is added in the body.

The response is parsed in the heartbeat function and the result is added to parameter resultName while generating an event.

A complete ESP project example is located in the ESP_Projects directory under [SOAP_LUA_NUMBERTOWORDS.xml](./ESP_Projects/SOAP_LUA_NUMBERTOWORDS.xml)

```lua
            local  url = "https://www.dataaccess.com/webservicesserver/NumberConversion.wso"
            local   connect_timeout = 3000
            local   data_timeout = 5000
                        
            function getResults(context,url)
                local   request = {}
                local   headers = {}
                math.randomseed(os.time())
                local   num = math.random(1,100)
                request["url"] = url
                request["method"] = "POST"
                request["connect-timeout"] = connect_timeout
                request["data-timeout"] = data_timeout
                headers["Content-Type"] = "text/xml; charset=utf-8"
                request["headers"] = headers
                request["body"]="<?xml version=\"1.0\" encoding=\"utf-8\"?><soap:Envelope xmlns:soap=\"http://schemas.xmlsoap.org/soap/envelope/\"><soap:Body><NumberToWords xmlns=\"http://www.dataaccess.com/webservicesserver/\"><ubiNum>".. num.."</ubiNum></NumberToWords></soap:Body></soap:Envelope>"
                request["tolua"] = true  -- convert json returned to lua table 
                
                local  response = sendHttp(request)
                local  status = response.status
                
                if status == 200 then return response.response
                else
                    logMessage(context.window,"HTTP request failed with status code  "..tostring(status),"info",debug.getinfo(1).currentline)
                    return false
                end     
            end  --getResults
           
            local   eventTime = os.time()       -- Time of last event generation
            local   eventInterval =  5          -- Generate events every 30 seconds
           
            function hb(context)  -- heartbeat function 
                
                local current = os.time()
                if (current - eventTime >= eventInterval )
                then
                    local events = {}      
                    local   guid = getGuids()[1]  -- base key for events 
                    local   counter = 1

                    local   data = getResults(context,url)
                    if data then -- results are available 
                       print("Generating event")    
                       local e = {}   -- create new event 
                       e.id = guid..":"..tostring(counter)
                       e.deviceid = "TestServer"
                       e.timestamp = os.time()
                       e.resultName = tostring(data.Envelope.Body.NumberToWordsResponse.NumberToWordsResult)
                       events[counter] = e 
                       eventTime = os.time()
                    else   events = nil 
                    end     
                return (events)    
                end
            end
            function create(data,context)
                return nil
            end
```

Once you run the project the result will be shown as below:


![image](Images/luasoap.PNG)

## Summary

Lua is a very powerful addition to the ESP tool bag.  Using these send HTTP examples it is possible to start collecting and processing data today.     

## Contributing

This project does not accept contributions.

## License

> This project is licensed under the [Apache 2.0 License](LICENSE).

## Additional Resources

**Required**. Include any additional resources that users may need or find useful when using your software. Additional resources might include the following:

* [Lua Pattern Matching](https://riptutorial.com/lua/example/20315/lua-pattern-matching)

* [SAS Event Stream Processing Documentation](https://go.documentation.sas.com/doc/en/espcdc/v_027/espcreatewindows/p0yj92wgv3ssyyn1syatsh9l1t74.htm#n0ppam7yzjg4b3n1jpn3igkaywt5)

* [Passing Lua Log Messages to the ESP Server](https://go.documentation.sas.com/doc/en/espcdc/v_027/espcreatewindows/p0yj92wgv3ssyyn1syatsh9l1t74.htm#p1jx8wo8si4xijn1p8lr88kd080l) 

  
