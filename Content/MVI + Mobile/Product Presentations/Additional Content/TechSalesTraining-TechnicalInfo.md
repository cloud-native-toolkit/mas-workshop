# System Properties
## Global System Properties
### maximo.mobile.fetch.timeout
Default 30000 (30 seconds). This controls the timeout for downloading data per <b>page</b> from the REST API. This is an important distinction as customers might be downloading thousands or millions of assets but we download them 1000 records at a time by default in mobile. Each data source can configure a different page size. 
<br /><br /><b>NOTE:</b> We currently have a performance issue with plussgeojson attribute on various tables (including ASSET & LOCATION) if the customer has Spatial. This attribute is built each time it's requested from ESRI data which makes requests that would take 2-4 seconds take  minutes. You will need to increase this limit or change the page size on those data sources if you have Spatial currently. 

### maximo.mobile.ldap.isForm
Default disabled (0). Customers need to set this to a 1 if they are using form based LDAP. The easy way to know if they're using basic auth vs form for LDAP is whether they see the traditional Maximo login screen. If they are presented a normal login screen and are utilizing LDAP then they are using form based.

Other authentication mechanisms (SAML & native) are handled automatically by the app provided they are on Maximo Mobile 8.6 or higher.

## Technician System Properties
### maximo.mobile.completestatus
Default COMP. Added in 8.7. This controls the status Maximo Mobile changes the WO when a user completes the work. For customers that need a different status (including synonyms of earlier statuses such as INPRG), they can change this to the desired status. 
<br /><br /><b>NOTE:</b> Customers will always need to manually filter out the status on their mobile-qbe-filter for removing the work from the list if it's a synonym of INPRG or earlier statuses. 
<br /><b> NOTE 2: </b> Until 8.7 IFIX 02, there is an issue when the status is a synonym of INPRG or earlier statuses. Users will not be redirected to the list page automatically.

### maximo.mobile.statusforphysicalsignature
Default null. Added in 8.7, but is available as a state variable in 8.6. This prompts the user to record their physical signature (not e-sig) when the WO is changed to the defined status. The resulting signature will be stored as an attachment.

### maximo.mobile.usetimer
Default enabled (1). Added in 8.7. When a user starts work we typically also start a timer for them. Disabling this will make it so a timer will not be started.

### mxe.mobile.travel.prompt
Default disabled (0). Added in 8.5. When enabled and the work is outside the radius defined in mxe.mobile.travel.radius system property, we'll prompt the user to "Start Travel" instead of "Start Work". This allows them to record travel time to get to the WO.

### mxe.mobile.travel.radius
Default 1. Added in 8.5. When mxe.mobile.travel.prompt is enabled this controls the distance when the Start Travel is presented. This is in miles for US/UK and KM for everywhere else. 
<br/><b>NOTE:</b> It is based on the country attribute defined on the person record. If this isn't set or not set exactly to US or UK we will use KM. 

### mxe.mobile.travel.navigation
Default disabled (0). Added in 8.5. When enabled and mxe.mobile.travel.prompt is enabled this will open the target location in a mapping application for navigation. In 8.8 this is planned to be enhanced to give the system administrator control of the application utilized for each platform. 

## Inspection System Properties
### mxe.app.inspection.UpdatePendingResults
Default enabled (1). Added in 8.6. This was added as part of the Maximo Mobile enhancements but occurs in core Maximo. When enabled and a new inspection form revision is approved, any inspection results in a PENDING status are updated with the changes. This is useful when you create WOs (typically via PMs) in advance. 

### mxe.app.workorder.InspectionBatchRecord
Default disabled (0). This was added as part of the Maximo Mobile enhancements but occurs in core Maximo. Enabling this will cause an additional inspection result to be created for the top level WO. This is how we supported batch inspections in earlier versions of Maximo. With it disabled we will not create the additional inspection result record. Users in Maximo Mobile can create their own ad-hoc batch inspections. 

### mxe.app.workorder.StatusToCreateInspection
Default WAPPR. Added in 8.6. This was added as part of the Maximo Mobile enhancements but occurs in core Maximo. Previously inspection results were created automatically as soon as the inspection form was associated to the work order. This allows you to delay it until the WO progresses to a desired status (such as APPR).
<br /><br /><b>NOTE:</b> PMs generate in a WSCH status by default and will most likely never hit a WAPPR status. In those scenarios the inspection result will not be generated. Customers should adjust this to the appropriate statuses for their organization.

### mxe.mobile.inspection.features.multiselect
Default enabled (1). Added in 8.7. This allows a customer to define a question as supporting a user selecting more than a single value (IE select all that apply). This is only available in the Graphite based application. We will not display multiselect in the Conduct an Inspection work center. 

### mxe.mobile.inspection.features.signature
Default enabled (1). Added in 8.7. This allows a customer to define a question as requiring a physical signature which will be stored as an attachment. This is only available in the Graphite based application. We will not display physical signature in the Conduct an Inspection work center. 

### mxe.mobile.inspection.mobilefeatures
Default enabled (1). Added in 8.6. This impacts a few settings such as being able to create conditions on numeric response values (IE require a response for a question when the value in a different question is > 5).

<br />

# Web vs Mobile Differences
One of the benefits of the new Graphite platform is it can be utilized to build mobile applications and web applications from a single code base. This allows us to replace our Anywhere & Work Center platforms without having to write two entirely separate applications. While the mobile and web version of an application function similarly, there are some important differences in how the application operates.

## mobile-qbe-filter
This is only used in Mobile and is utilized on data sources to provide an additional filter locally on the data source. When changes occur to the record and the transaction may not have synchronized to Maximo yet, this helps ensure the record is hidden from the data source and thus the UI. For example, we utilize this to filter out work orders in a COMP status so that the record is immediately removed from the list when they complete it even if they are disconnected.

## Status Changes

### Web
For web requests, we request the allowedstates from the REST API in the oslc.select. For stateful records, this allows us to see what status changes Maximo will allow for the record. This is similar to executing the change status in Maximo and seeing the list filtered based on security, conditions tied to the domain values, and of course the current status (can only go from COMP to CLOSE or a synonym of COMP for example).

### Mobile
For mobile requests, we retrieve the allowedstates but we do not utilize it. This is because changes to the record (going to INPRG for example) would change the allowedstates and we can't be sure that we've retrieved it because we might be disconnected. 

Up until 8.7 for TECHMOBILE, we utilized a static matrix in the AppController.js that stated which internal values were allowed for each status change. For example, when COMP you could only go to another synonym of COMP or CLOSE.

Customers wanted some additional control over the status flow and we don't support modifying the AppController which prevented them from doing this. In 8.7 IFIX 01 we added a new state variables to TECHMOBILE to provide more control without modifying our AppController.

customAllowedStates - This is a user modifiable matrix. If you do not want a user to go from APPR->WAPPR for example, even though this is allowed in Maximo, you can remove the mapping. Because this is an XML document any reserved characters (double quote for example) needs to be replaced with the XML equivalent. 

## Searching/Filtering

### Web
Attributes that are defined as searchable="true" are provided as searchAttributes in the REST API request and the value provided for the search is set as the oslc.searchTerms. The REST API then builds a where clause searching on each of those attributes with an OR statement inbetween. 

Filtering via the REST API introduces a few differences over mobile. Attributes that are TEXT search enabled for example will utilize the text search functionality on the database platform. Certain characters that are reserved for text search (such as - in Oracle) will cause the search to function differently.

### Mobile
The attributes defined as searchable on the data source are indexed in the local database. We utilize a local search of all attributes as wildcard search so issues around text search for example won't occur on the mobile device. All searches are done against the local database, even if good connectivity exists.

## Data Retrieval

### Web
We make requests in a way most organizations are familiar with. We'll make a GET request via the REST API with the oslc.select, oslc.where, etc. query parameters in the request. Reviewing the web server logs and the Maximo logs you'll see the exact request being made. This makes debugging API requests easier.

From a performance perspective, analyzing the network requests utilizing Developer Tools here is useful because you'll be able to see exactly what was requested and how long each request took. You'll be able to then investigate root cause. 

### Mobile
Due to how we manage data sources (see next section), we hit limitations with the length of the URL. That required us to change our approach to retrieving data. We now utilize a x-www-form-urlencoded POST request to the REST API to retrieve data. Things that would normally be query parameters are provided as form items in key value pairs (oslc.select, oslc.where, etc.). 

This makes debugging a request more difficult because the webserver & Maximo logs will not list the body of the request. Utilizing tools like Wireshark on the server might be required to debug.

## Data Sources
One of the biggest differences is how we handle data sources. 

### Web 
In the web version of the application, we retrieve each data source independently. That means we make a separate network request for each data source. Each data source therefore only impacts itself. A poor performing API request might prevent you from utilizing a particular function (seeing other WOs for the asset/location for example) but would otherwise not impact the rest of the application.

### Mobile
In the mobile version of the application, we will retrieve multiple data sources together into a single request. We essentially concatenate the oslc.select for all data sources sharing that object structure to make the request. If a user can see Work Orders in the web view of the application but not the mobile version, the issue might be related to one of these child data sources impacting the performance of the request.

As an example, our method to retrieve other WOs against the same Asset/Location had an issue that is mostly resolved in 8.7 IFIX 01. If you had an Asset/Location with a lot of open WOs that could cause your data download to fail.

## Response Properties
For those unaware, when you submit a record to Maximo via the REST API you can provide a properties header with a comma separated list of attributes to be returned back. This helps ensure that things that only get set on the server side (automation script for example) get updated locally as well. While sort of linked to dependent data sources, it seemed appropriate to separate this into its own topic.

### Web
Each change to a data source retrieves only the attributes defined in that data source. Phrashed differently, we set the properties header equal to what the oslc.select would be for that data source.

### Mobile
By default, all attributes (including from other data sets associated to that object structure) are retrieved in the request. This can be problematic from a performance perspective if some of the related data is poor performing. But more importantly, this can also introduce issues due to the header length limit. When the header limit is exceeded, the transactions will show as queued with no visible indicator to the user today. In the Mobile logs of the device you'll see a 400 error and in the web server logs you should see a more detailed error messages. For IHS, you can review this for how to extend it: https://www.ibm.com/support/pages/request-header-field-size-exceeds-limit-web-server

To avoid this issue, when we work with data sources we can provide an option object that supports defining a responseProperties. Providing the desired attributes to be returned will be utilized on the request. For example, we added this to the FailureDetailsPageController to avoid this header issue.

```javascript
    let option = {
      responseProperties: workorderDS.baseQuery.select
    };
    await workorderDS.update(item,option);
```