# Maximo Tech Seller Customizations
## By Steven Shull<br/><br/>


### 1) Restrict action based on security permissions
**Use Case**: An organization may want to allow a user to take an action but only if given proper authorization inside of Maximo. In this scenario we will restrict a user's ability to create a follow up work order based on them having the appropriate permission on the object structure in Maximo.

**Solution Summary**: Maximo Mobile supports tying a sigoption to controls (such as the button) which will disable the button when the user is not granted the permission. Security permissions are managed by APPNAME/OSNAME.OPTIONNAME. In this case, because we're using MXAPIWODETAIL object structure with the CREATEWO sigoption, we'll reference that permission. <br /><br />

**Original**: 
```xml
<button disabled="{woDetailRelatedWorkOrder.item.wonum ? false : true}" icon="carbon:add" id="m3nvj" kind="ghost" label="Create follow-up work" loading="{page.state.loadingfollowup === true}" on-click="createRelatedAndFollowUpWo" on-click-arg="{{'item': woDetailRelatedWorkOrder.item, 'datasource': woDetailRelatedWorkOrder}}" padding="false"/>
```
**After Change**: 
```xml
<button disabled="{woDetailRelatedWorkOrder.item.wonum ? false : true}" sigoption="MXAPIWODETAIL.CREATEWO" icon="carbon:add" id="m3nvj" kind="ghost" label="Create follow-up work" loading="{page.state.loadingfollowup === true}" on-click="createRelatedAndFollowUpWo" on-click-arg="{{'item': woDetailRelatedWorkOrder.item, 'datasource': woDetailRelatedWorkOrder}}" padding="false"/>
```
<br /><br />


### 2) Display Hazard & Precaution Information
**Use Case**: Maximo Mobile doesn't currently display Hazards & Precautions but these can be critical for organizations to ensure their technicians are taking proper safety precautions. For simplicity, the technician will not be able to modify these values.

**Solution Summary**: Because the MXAPIWODETAIL isn't allowed to be modified, we'll reference these in the app.xml as separate data sources. We'll use the relationship WOHAZARD to get to the WO Hazards and WOHAZARDPREC to get the Precautions under each Hazard. We'll use the report labor for inspiration on how to group these under the appropriate section.

**Example Code- Data Source**: 
```xml
<!-- Add to the id="woDetailResource" resource, below the </schema> tag -->
        <maximo-datasource depends-on="woDetailResource" id="emxWoHazards" relationship="wohazard" selection-mode="none" notify-when-parent-loads="true" object-name="wohazard">
          <schema>
              <attribute name="wohazardid" unique-id="true"/>
              <attribute name="hazardid"/>
              <attribute name="hazardtype"/>
              <attribute name="description--hazarddescription"/>
              <attribute name="rel.wohazardprec{precautionid,description--precautiondescription}"/>
          </schema>
        </maximo-datasource>
```
**Example Code- Display**: 
```xml
<!-- Add wherever you want to display. I added it above id="kxgm8" -->
      <box>
        <border-layout padding="{app.state.screen.size === 'sm' || app.state.screen.size === 'md' ? 'false' : 'true'}" width="100%">
          <top>
            <data-list datasource="emxWoHazards" empty-row-style="true" empty-set-string="No hazards identified." margin="false" show-search="false" >
              <list-item child-attribute="wohazardprec">
                <adaptive-row>
                    <adaptive-column large-horizontal-align="start" large-width="25" medium-horizontal-align="start" medium-width="25" small-horizontal-align="start" small-width="25">
                        <box direction="column" fill-parent="true" vertical-align="center">
                          <label label="{item.precautionid}"/>
                        </box>
                    </adaptive-column>
                    <adaptive-column large-horizontal-align="start" large-width="75" medium-horizontal-align="start" medium-width="75" small-horizontal-align="start" small-width="75">
                        <box direction="column" fill-parent="true" vertical-align="center">
                          <label label="{item.precautiondescription}"/>
                        </box>
                    </adaptive-column>                    
                </adaptive-row>
              </list-item>
              <list-item>
                <box>
                  <icon-group placeholder-label="{item.hazardid} {item.hazarddescription}" primary-icon="carbon:warning--alt"/>
                </box>
              </list-item>
            </data-list>
          </top>
        </border-layout>
      </box>
```
<br /><br />

### 3) Filter Allowed WO Statuses
**Use Case**: Maximo Mobile displays to users all WO statuses allowed on the record. An organization wants to restrict this to a subset of statuses for all users in the mobile application. For example, you may not want users to be able to change the status to COMP but a synonym of COMP (such as WREVIEW). Maximo Mobile loads data from either synonymdomain (mobile app) or getting allowedstates for each Work Order from the REST API (web) into a JSON data source called dsstatusDomainList. Filtering the synonym domain data source is an issue because Maximo Mobile needs to have all statuses to know what state the WO is in to allow/disallow certain changes. Since modifying the synonym domain data source wouldn't support the web view and impacts core functionality we need an alternative method to restrict statuses.

**Solution Summary**: We will create an “onAfterLoadData” method in our AppCustomizations.js. This method has two arguments (dataSource & items) that are passed in. This will fire for each data source so we check the dataSource name to ensure it's the one we need. We generate a separate array of items we want to remove from the list by filtering the array with a function we define called woStatusFilter. For each item in the list it will execute the function (providing just a single value) that we then compare based on our hardcoded array of allowed values. Because we want to return items we want to delete (not keep), we return the inverse by using the exclamation mark (!). We then call the dataSource.deleteItems to remove all items in our filtered list. <br /><br />**NOTE**: You should NOT call deleteItems on most data sources as that will flag the record for deletion in Maximo. Since this data source is a temporary JSON data source on the device, we're able to safely remove records from it without impacting data in Maximo.

**Example Code- AppCustomizations.js**:
```javascript
   woStatusFilter(item)
   {
	 // Because this will return items to delete, we need the "inverse" of our desired results.
     return !['WMATL','INPRG','COMP'].includes(item.value);
   }
   async onAfterLoadData(dataSource,items)
   {
	   if (dataSource && dataSource.name==="dsstatusDomainList")
	   {
			// Ensure we have at least one record
			if (items && items.length>0)
			{
				let filteredItems=items.filter(this.woStatusFilter);
				await dataSource.deleteItems(filteredItems);
			}
	   }
   }
``` 
<br /><br />

### 4) Set Work Order Owner Group on new WOs
**Use Case**: In some organizations, the technician creating the WO will be responsible for assigning ownership of the WO. 

**Solution Summary**: We will add the ownergroup field and provide a lookup for the user to utilize. This requires adding the ownergroup attribute to the dsCreateWo data source (for storing the value), a new data source for retrieving our person groups, a lookup for displaying the values from our new data source, and finally a JavaScript method for taking the selected value from the lookup and setting it on our data source. 

**Add to id="dsCreateWo"**:
```xml
<attribute name="ownergroup" />
```

**New Data Source**:
```xml
  <!-- Add below id="synonymdomainData" -->
  <maximo-datasource id="persongroupLookupDS" lookup-data="true" object-structure="mxapipersongroup" selection-mode="single">
    <schema>
      <attribute name="persongroup" searchable="true" />
      <attribute name="description" searchable="true" />
    </schema>
  </maximo-datasource>
```

**New Lookup**
```xml
<!-- Add inside dialogs id="wrgyq"-->
<lookup-with-filter id="ownergroupLookup" width="99" datasource="persongroupLookupDS" show-search="true" on-item-click="selectOwnerGroup" lookup-attributes="{['persongroup','description']}" show-count="true" lookup-heading="Owner Group" />
```

**Display Input**
```xml
<!-- Add above id="a5ezy"-->
<box direction="row" children-sizes="50" fill-parent="true" fill-child="true" padding-bottom=".5" padding-top=".5" >
    <smart-input label="Owner Group" theme="dark" placeholder="Owner Group" value="{dsCreateWo.item.ownergroup}" select-lookup-attribute="persongroup" lookup="ownergroupLookup" enable-lookup-buttongroup="true" />
 </box>  
```

**JavaScript Function - AppCustomization.js**
```javascript
   async selectOwnerGroup(event) {
     let dsCreateWo = this.app.findDatasource("dsCreateWo");
     dsCreateWo.item["ownergroup"]=event.persongroup;
   }
```