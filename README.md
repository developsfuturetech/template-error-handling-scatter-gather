﻿# Error handling of Scatter Gather in the Salesforce and Database Aggregation template ###

This application illustrates how to handle custom exceptions through an example of integrating Salesforce and a MySQL Database.

### Assumptions ###

This document assumes that you are familiar with Mule and the [Anypoint™ Studio interface](http://www.mulesoft.org/documentation/display/current/Anypoint+Studio+Essentials). To increase your familiarity with Studio, consider completing one or more [Anypoint Studio Tutorials](http://www.mulesoft.org/documentation/display/current/Basic+Studio+Tutorial). Further, this example assumes that you have a basic understanding of [Mule flows](http://www.mulesoft.org/documentation/display/current/Mule+Application+Architecture), [Mule Global Elements](http://www.mulesoft.org/documentation/display/current/Global+Elements) and Scatter Gather component [Scatter Gather](http://www.mulesoft.org/documentation/display/current/scatter-gather)
This document describes the details of the example within the context of Anypoint Studio, Mule ESB’s graphical user interface.

### Use Case ###

This application aggregates users from a Salesforce Instance and a Database, compares the records to avoid duplication and then tranfers it to a CSV file which is then sent as an attachment via email. This application also shows you how to define and handle exceptions such as issue with login, field mismatch between the different source systems and so on.


### Set Up and Run the Example ###

Complete the following procedure to create, then run this example in your own instance of Anypoint Studio. 

There are certain conditions that need to be met in order for this application to work as expected. They have been explained below.**Failling to do so could lead to unexpected behavior of the template.**

**Note:** This particular Anypoint Template ilustrate the aggregation use case between SalesForce and a Database, thus it requires a DB instance to work.
The Anypoint Template comes package with a SQL script to create the DB table that uses. 
We expect the user of the application to utilize the script to create a table with the appropriate schema.
The SQL script file can be found in [src/main/resources/sfdc2jdbc.sql] (../master/src/main/resources/sfdc2jdbc.sql)

To trigger the use case you just need to make a call to the local http endpoint at the port you configured in your file. If this is, for instance, `9090` then you should hit: `http://localhost:9090/synccontacts` and this will create a CSV report and send it via email.

### Application Configuration ###
To make the application run, it's required to configure the different end points involved. The configuration files is located for the chosen environment in the [src/main/resources/mule.env.properties]
The following is an example to show you what vales are expected from the user during the configuration.

http.port `9090` 

#### SalesForce Connector configuration for company A
+ sfdc.username `bob.dylan@orga`
+ sfdc.password `DylanPassword123`
+ sfdc.securityToken `avsfwCUl7apQs56Xq2AKi3X`
+ sfdc.url `https://login.salesforce.com/services/Soap/u/32.0`

#### Database Connector configuration
+ db.jdbcUrl `jdbc:mysql://localhost:3306/sfdc2jdbc?rewriteBatchedStatements=true&user=root&password=password`

#### SMTP Services configuration
+ smtp.host `smtp.gmail.com`
+ smtp.port `587`
+ smtp.user `exampleuser@gmail.com`
+ smtp.password `ExamplePassword456`

#### Mail details
+ mail.from `exampleuser@gmail.com`
+ mail.to `woody.guthrie@gmail.com`
+ mail.subject `SFDC Users Report`
+ mail.body `Users report comparing users from SFDC Accounts`
+ attachment.name `OrderedReport.csv`

### How it Works ###
The application is structured in the several files based on the functional aspect.

#### default_endpoints_xml
This is the file where you will found the inbound and outbound sides of your integration app.
This Anypoint Template has an [HTTP Inbound Endpoint](http://www.mulesoft.org/documentation/display/current/HTTP+Endpoint+Reference) as the way to trigger the use case and an [SMTP Transport](http://www.mulesoft.org/documentation/display/current/SMTP+Transport+Reference) as the outbound way to send the report.

Trigger Flow
**HTTP Inbound Endpoint** - Start Report Generation
`${http.port}` is set as a property to be defined either on a property file or in CloudHub environment variables.
The path configured by default is `generatereport` and you are free to change for the one you prefer.
The host name for all endpoints in your CloudHub configuration should be defined as `localhost`. CloudHub will then route requests from your application domain URL to the endpoint.

Outbound Flow
**SMTP Outbound Endpoint** - Send Mail

Both SMTP Server configuration and the actual mail to be sent are defined in this endpoint.
This flow is going to be invoked from the flow that does all the functional work: *mainFlow*, the same that is invoked from the Inbound Flow upon triggering of the HTTP Endpoint.

#### default_business_logic_xml
Functional aspect of the Anypoint Template is implemented on this XML, directed by one flow responsible of conducting the aggregation of data, comparing records and finally formating the output, in this case being a report.
The *mainFlow* organises the job in three different steps and finally invokes the *outboundFlow* that will deliver the report to the corresponding outbound endpoint.
This flow has Exception Strategy that basically consists on invoking the *defaultChoiseExceptionStrategy* defined in *errorHandling.xml* file.

Gather Data Flow
Mainly consisting of two calls (Queries) one to SalesForce and one to a DB and storing each response on the Invocation Variable named *usersFromOrgA* or *usersFromDB* accordingly.

[Scatter Gather](http://www.mulesoft.org/documentation/display/current/Scatter-Gather) is responsible for aggregating the results from the two collections of Users.
Criteria and format applied:

Scatter Gather component implements an aggregation strategy that results in List of Maps with keys: **Name**, **Email**, **IDInA**, **UserNameInA**, **IDInB** and 
**UserNameInB**.
Users will be matched by an email, that is to say, a record in both SFDC organisations with same mail is considered the same user.

Format Output Flow
[Java Transformer](http://www.mulesoft.org/documentation/display/current/Java+Transformer+Reference) responsible for sorting the list of users in the following order:

1. Users only in Org A
2. Users only in Org B
3. Users in both Org A and Org B

All records ordered alphabetically by mail within each category.
If you want to change this order then the *compare* method should be modified.

CSV Report [DataWeave](http://www.mulesoft.org/documentation/display/current/DataWeave+Reference+Documentation) transforming the List of Maps in CSV with headers **Name**, **Email**, **IDInA**, **UserNameInA**, **IDInB** and **UserNameInB**.
An [Object to string transformer](http://www.mulesoft.org/documentation/display/current/Transformers) is used to set the payload as an String. 

#### default_error_handling_xml
This is the right place to handle how your integration will respond to different types of exceptions. 
This file holds a [Choice Exception Strategy](http://www.mulesoft.org/documentation/display/current/Choice+Exception+Strategy) that is referenced by the main flow in the business logic.

If there is an error in *gatherDataFlow* while obtaining data for aggregation, it will be first handled in *UserMergeAggregationStrategy* class. Scatter-gather component uses this custom aggregation strategy class which overrides its default aggregation strategy to perform aggregation of the retrieved data with desired error handling. For more details click [here](https://docs.mulesoft.com/mule-user-guide/v/3.7/scatter-gather#scatter-gather-behavior-and-exceptions). After that the error will be routed to errorHandling.xml.

In the case that no data was obtained from the end points, the appropriate  error will be thrown from the *UserMergeAggregationStrategy* flow. Based on the flow definitions in the errorHandling.xml file, the error will then be routed to the correct catch block under the *defaultChoiceExceptionStrategy*. Each error specified in the catch block can be treated in a different way. 

In this application, there are three different kinds of errors specified:

*LoginFault* Exception - occurs in the case of incorrect Salesforce credentials
*InvalidFieldFault* Exception - occurs in the case that a newly added column name in the Salesforce query is not present in the specified table (could be used for the custom defined fields)
*Exception* - any other exception except the previous two

New exceptions can be added and caught in the errorHandling.xml file.

Data obtained from just one route while the other source fails is considered a recoverable scenario. No error message is logged and the data is processed. This can be changed in the *UserMergeAggregationStrategy* class.

All included connectors support [Reconnection Strategies](https://docs.mulesoft.com/mule-user-guide/v/3.7/configuring-reconnection-strategies). Reconnection Strategies specify how a connector behaves when its connection fails. In this template all connectors have standard reconnection strategy specified to 5 reconnection attempts with frequency 5000ms. You can change reconnection strategy in connector configuration window by selecting tab *Reconnection*.

### Api calls ###
SalesForce imposes limits on the number of API Calls that can be made. Therefore calculating this amount may be an important factor to consider. User Anypoint Template calls to the API can be calculated using the formula:

***1 + UsersToSync + UsersToSync / CommitSize***

Being ***UsersToSync*** the number of Users to be synchronized on each run. 

The division by ***CommitSize*** is because by default, for each Upsert API Call, Users are gathered in groups of a number defined by the Commit Size property. Also consider that this calls are executed repeatedly every polling cycle.	

For instance if 10 records are fetched from origin instance, then 12 api calls will be made (1 + 10 + 1).

### Documentation ###

Studio includes a feature that enables you to easily export all the documentation you have recorded for your project. Whenever you want to share your project with others outside the Studio environment, you can export the project's documentation to print, email or share online. Studio's auto-generated documentation includes:

- A visual diagram of the flows in your application
- The XML configuration which corresponds to each flow in your application
- The text you entered in the Notes tab of any building block in your flow

Follow [the procedure](http://www.mulesoft.org/documentation/display/current/Importing+and+Exporting+in+Studio#ImportingandExportinginStudio-ExportingStudioDocumentation) to export auto-generated Studio documentation.
