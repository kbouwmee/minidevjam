# Lab Appendix 2- Create a Mashup with APIs

As a member of the API team, you would like to aggregate multiple APIs, in this case the Order API and Google Geolocation API, into a single interface for ease of consumability for your App Developers when searching for employees located within a certain distance of a given location.

The goal is to have the proxy perform a geolocation query against our Order data collection to return results within a certain radius of a zip code.
API BaaS supports the ability to retrieve entities within a specified distance of any geocoordinate based on its location property:

```
location within {distance_in_meters} of {latitude},{longitude}
```

As you can see, you need to provide the latitude and longitude information to perform the query.
For mobile applications meant for smartphones, obtaining geocode information is easy and can be provided directly as part of an API call. For this lesson, assume that this API interface is being created for devices and applications that cannot easily provide the geocoordinate information, but simply requests the user to provide the zip code. In such a situation, the first thing is to obtain the geo-coordinates for the zip code provided before doing further processing. Below are the high level steps to implement this in the proxy:
* Retrieve the zipcode and radius from the request query parameters
* Use the zipcode as an input parameter to call an external service that converts the zipcode to the geo-coordinates
* Extract the latitude and longitude geo-coordinates information from the response of the external service call
* Use the geo-coordinates to create the geo-location query
* Add the location query as a query parameter before the target BaaS service is invoked

A pictorial representation of the logic is depicted below:

![Image](images/lab_appendix2/image00.png) 

For the service callout to convert the zipcode to the geocoordinate, you will use the [Google GeoCoding API](https://developers.google.com/maps/documentation/geocoding/).

##How can Apigee Edge help?
Apigee Edge enables you to design API behavior by using the out of the box policie'. A policy is like a module that implements a specific, limited management function. Policies are designed to let you add common types of management capabilities to an API easily and reliably.

In this lab we will see how you can extend an existing API by aggregating it with another API and creating a single interface for your App Developers to consume. In this example, we will be combining data from our Order API with the Google Weather to have a single response of the order information and weather for their location. We will discover and use the following types of policies:

**Traffic Management Policies** in the traffic management category enable you to control the flow of request and response messages through an API proxy. These policies support both operational- and business-level control. They give you control over raw throughput, and can also control traffic on a per-app basis. Traffic management policy types enable you to enforce quotas, and they also help you to mitigate denial of service attacks.

**Mediation Policies** in the mediation category enable you to actively manipulate messages as they flow through API proxies. They enable you to transform message formats, from XML to JSON (and vice-versa), or to transform one XML format to another XML format. They also enable you to parse messages, to generate new messages and to change values on outbound messages. Mediation policies also interact with basic services exposed by API Services, enabling you to retrieve data about apps, developers, security tokens, and API products at runtime.

**Security Policies** in the security category support authentication, authorization, as well as content-based security.

**Extension Policies** in the extension category enable you to tap into the extensibility of API Services to implement custom behavior in the programming language of your choice.

##Pre-requisites
* An existing API proxy

##Instructions
###Select existing Orders API
* Go to [https://apigee.com/edge](https://apigee.com/edge) and log in. This is the Edge management UI. 
* Select **Develop → API Proxies** in the side navigation menu.

![Image](images/lab_appendix2/image01.png) 

* Click on the proxy you created in the [Lab 1 - Adding a new API Specification](lab1.md).

![Image](images/lab_appendix2/image02.png) 

* Click on the **Develop** tab

###Use an Assign Message Policy to prepare the service callout request

* Click on **+ Step**

![Image](images/lab_appendix2/image03.png) 

Scroll down the policy list and select Assign Message and update the default display name to Create Geocoding Request. Then click on Add button.

![Image](images/lab_appendix2/image04.png) 

* Modify the policy to reflect a request with the appropriate query parameters for the Google Geolocation API.

```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<AssignMessage async="false" continueOnError="false" enabled="true" name="Create-Geocoding-Request">
    <DisplayName>Create Geocoding Request</DisplayName>
    <AssignTo createNew="true" type="request">GeoCodingRequest</AssignTo>
    <Set>
        <QueryParams>
            <QueryParam name="address">{request.queryparam.zipcode}</QueryParam>
            <QueryParam name="region">US</QueryParam>
            <QueryParam name="sensor">false</QueryParam>
        </QueryParams>
        <Verb>GET</Verb>
    </Set>
    <!-- Set variables for use in the flow -->
    <AssignVariable>
        <Name>zipcode</Name>
        <Ref>request.queryparam.zipcode</Ref>
    </AssignVariable>
    <AssignVariable>
        <Name>radius</Name>
        <Value>0</Value>
        <Ref>request.queryparam.radius</Ref>
    </AssignVariable>
</AssignMessage>
```

Here's a brief description of the elements in this policy. You can read more about this policy in Assign Message policy.

**<AssignMessage name>** - Gives this policy a name. The name is used when the policy is referenced in a flow.

**<AssignTo>** - Creates a named variable called ’GeoCodingRequest’of type ‘Request’. This variable encapsulates the request object that will be sent by the ServiceCallout policy.

**<Set><QueryParams>** - Sets the query parameters that are needed for the service callout API call. In this case, the Google Geocoding API needs to know the location, which is expressed with a zipcode. The API calling client supplies this information, and we simply extract it here. The region and sensor parameters are by the API, and we just hardcode it to certain values here.

**<Verb>** - In this case, we are making a simple GET request to the API.

**<AssignVariable>** - zipcode and radius are new variables being created to store values being passed to the API. In this example, the variables will be accessed later in the proxy flow.

**Note:** The properties associated with the ‘Assign Message’ policy could have been modified using the ‘Property Inspector’ panel that’s presented in the ‘Develop’ tab on the right. Any changes made in the ‘Code’ panel are reflected in the ‘Property Inspector’ panel and vice-versa. We will use the ‘Property Inspector’ panel to set properties for some of the policies as the lesson progresses.

### Use Service Callout Policy to invoke the Google GeoCoding API


* Click on **+ Step**

![Image](images/lab_appendix2/image05.png) 

Scroll down the policy list and select Service Callout and update the default display name to Call Geocoding API, select HTTP and then enter the HTTP Target with the following 
URL:  [http://maps.googleapis.com/maps/api/geocode/json](http://maps.googleapis.com/maps/api/geocode/json)

![Image](images/lab_appendix2/image06.png) 

Then click on **Add** button.

* Update the **Request** variable from myRequest to **GeoCodingRequest** and also update the **Response** variable from calloutResponse to **GeocodingResponse**.

![Image](images/lab_appendix2/image07.png) 

**<Request variable>** - This is the variable ‘GeoCodingRequest’ that was created in the AssignMessage policy in the previous step. It encapsulates the request going to the Google Geocoding API.

**<Response>** - This element names a variable ‘GeoCodingResponse’ in which the response from the Google Geocoding API will be stored. As you will see, this variable will be accessed later by the ExtractVariables policy.

**<HTTPTargetConnection><URL>** - Specifies the target URL to be used by the service callout - in this case the URL of the Google Geocoding API: http://maps.googleapis.com/maps/api/geocode/json

### Use Extract Message Policy to parse the service callout response
* Click on **+ Step**

![Image](images/lab_appendix2/image08.png) 

Scroll down the policy list and select **Extract Variables** and update the default display name to **Extract Geocodes**

![Image](images/lab_appendix2/image09.png) 

Then click on **Add**.

* Update the policy to parse the **GeoCodingResponse** and store the results in variables.

```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ExtractVariables async="false" continueOnError="false" enabled="true" name="Extract-Geocodes">
    <DisplayName>Extract Geocodes</DisplayName>
    <Source>GeoCodingResponse</Source>
    <VariablePrefix>geocodeResponse</VariablePrefix>
    <JSONPayload>
        <Variable name="latitude">
            <JSONPath>$.results[0].geometry.location.lat</JSONPath>
        </Variable>
        <Variable name="longitude">
            <JSONPath>$.results[0].geometry.location.lng</JSONPath>
        </Variable>
    </JSONPayload>
</ExtractVariables>
```

Here's a brief description of the elements that were modified in this policy. You can read more about this policy in [Extract Variables policy](https://www.google.com/url?q=http://apigee.com/docs/api-services/reference/extract-variables-policy&sa=D&ust=1484850149928000&usg=AFQjCNEvSjBrn8xBtS20uEMnXVkmp3tGIA).
- Specifies the response variable ‘GeoCodingResponse’ that we created in the ServiceCallout policy. This is the variable from which this policy extracts data.
- The variable prefix ‘geocodeResponse’ specifies a namespace for other variables created in this policy. The prefix can be any name, except for the reserved names defined by the [Apigee Edge Platform's predefined variables](https://www.google.com/url?q=http://apigee.com/docs/api-platform/api/variables-reference&sa=D&ust=1484850149930000&usg=AFQjCNFJBZO91HDuovPquvIspytNr8VE2A).
- This element retrieves the response data that is of interest and puts it into named variables. In fact, the Google Geocoding API returns much more information than latitude and longitude. However, these are the only values needed for these lessons. You can see a complete rendering of the JSON in the [Google Geocoding API documentation](https://www.google.com/url?q=https://developers.google.com/maps/documentation/geocoding/&sa=D&ust=1484850149931000&usg=AFQjCNHRHGwYcNg8Py-7kW7HVu5LFmS5eg). The values of geometry.location.lat and geometry.location.lng are simply two of the many fields in the returned JSON object.

It may not be obvious, but it's important to see that ExtractVariables produces two variables whose names consist of the variable prefix (geocodeResponse) and the actual variable names that are specified in the policy. These variables are stored in the API proxy and will be available to other policies within the proxy flow, as you will see. The variables are: geocodeResponse.latitude & geocodeResponse.longitude

### Use the Javascript Policy to create the Location Query to send to the BaaS target endpoint

* Click on **+ Step**

![Image](images/lab_appendix2/image10.png) 

Scroll down the policy list and select **Javascript** and update the default display name to **Create Location Query**, select **Create New Script** and then name it **Create-Location-Query.js**

![Image](images/lab_appendix2/image11.png) 

* Select the newly created script file and add the following code:

![Image](images/lab_appendix2/image12.png) 

```
var latitude = context.getVariable("geocodeResponse.latitude"),
    longitude = context.getVariable("geocodeResponse.longitude"),
    radius = context.getVariable("radius");

// set default (0 meters)
radius = (radius === "") ? "0" : radius;

// set BaaS query
var baasQL = "location within " + radius + " of " + latitude + "," + longitude;
context.setVariable("baasQL", baasQL);
```

This Javascript code uses the ‘context’ object, which is part of the [Apigee Edge Javascript object model](http://apigee.com/docs/api-services/reference/javascript-object-model) to retrieve 3 variables - geocodeResponse.latitude, geoCodeResponse.latitude, radius - that were set by policies earlier in the flow.

It sets a default in case the variables are empty strings, creates a new query variable called ‘baasQL’ using the API BaaS query language syntax for a location query, and adds the ‘baasQL’ variable to the ‘context’ object to be used later in the flow by the Assign Message policy to set the query parameter before the API BaaS target endpoint is invoked.

You can read more about this policy in [Javascript policy](http://apigee.com/docs/api-services/reference/javascript-policy).


### Use the Assign Message Policy to add the Location Query to the query parameter before BaaS target endpoint invocation

* Click on + Step

![Image](images/lab_appendix2/image13.png) 

Scroll down the policy list and select **Assign Message** and update the default display name to **Set Query Parameters**

![Image](images/lab_appendix2/image14.png) 

Update the policy to include the **baasQL** as a query parameter and remove the zipcode and radius query parameters from the request.

```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<AssignMessage async="false" continueOnError="false" enabled="true" name="Set-Query-Parameters">
    <DisplayName>Set Query Parameters</DisplayName>
    <Remove>
        <QueryParams>
            <QueryParam name="zipcode"/>
            <QueryParam name="radius"/>
        </QueryParams>
    </Remove>
    <Set>
        <QueryParams>
            <QueryParam name="ql">{baasQL}</QueryParam>
        </QueryParams>
    </Set>
</AssignMessage>
```


Here's a brief description of the elements that were modified in this policy. You can read more about this policy in [Extract Variables policy](http://apigee.com/docs/api-services/reference/extract-variables-policy).
- Removes the query parameters (‘**zipcode**’ and ‘**radius**’) that were sent in the original client request to the API Proxy.
- Adds a new query parameter (‘**ql**’) with the variable **baasQL** providing the actual value. Note that the **baasQL** variable was set by the previous Javascript policy as part of the **context** object

### Testing the API Proxy with the location query after deploying changes

* Click on the **Save** button to save and deploy the changes to the API Proxy.

![Image](images/lab_appendix2/image15.png) 

* Go to the **Trace** tab and start a trace session by clicking the ‘Start Trace Session’ button

![Image](images/lab_appendix2/image16.png) 

* Using your browser or the Apigee REST Client, invoke the API with the following query parameter combinations and review the results being returned:
	*   zipcode=31721&radius=20000
	*   zipcode=31721&radius=500000
	*   No query parameters

**\*\*Note: radius is measured in meters**

Example URL: [http://apigeedemovideos-test.apigee.net/v1/ap_employees?zipcode=31721&radius=20000](http://apigeedemovideos-test.apigee.net/v1/ap_employees?zipcode=31721&radius=20000)
