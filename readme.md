<!-- TABLE OF CONTENTS -->
<details open="open">  
  <ol>
    <li>
      <a href="#about-the-project">Part 1</a>      
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#prerequisites">Prerequisites</a></li>
        <li><a href="#installation">Installation</a></li>
      </ul>
    </li>
    <li><a href="#usage">Usage</a></li>
    <li><a href="#roadmap">Roadmap</a></li>
    <li><a href="#contributing">Contributing</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
    <li><a href="#acknowledgements">Acknowledgements</a></li>
  </ol>
</details>



<!-- Part 1 -->
# Part 1

Item trader project is a web api project that provides endpoints for listing items to be used in trades. The project is built for an interactive UI (Native mobile apps or Web apps having backend or not) which will provide its users ability to trade their items.
The application consists of trade items and proposals. An usual flow would be :

* User lists several items to trade.
* Sees other trade items in the system. 
* Can give a proposal for one of the items.
* The requested items owner may reject or accept the proposal. And the proposal owner can cancel the proposal if it is not accepted or rejeected yet.
* If the proposal is accepted, the trade happens. Otherwise the proposed item will be available for trading again.
* After a successful trade happens, both of the items are delisted from available to trade items list.

The Item Trader API is built with .Net Core 3.1 and using a SQL Database. 
Clean architecture design is implemented at the application. So the API has Domain and Applicaiton Layers acting as Core. An Infrastructure and a presentation layer depends on the core.

I used Mediatr while implementing CQRS which helps a lot with its pipeline, request handler and event handler features. 
I also used FluentValidation for validating requests and AutoMapper for mapping Domain objects to DTOs. EntityFrameworkCore is used for accessing database.

<br/>
<br/>

<details open=open>
<summary> ## Authentication </summary>

For authentication and authorization, i used IdentityServer4 with .Net Core Identity. IdentityServer itself is not an authentication server so with .Net Core identity it also provides authentication by applying Open ID Connect and OAuth protocols.

Authentication server and Item Trader API are handled as completely separated domains. The only thing these two domains share is the database. But as Domain Driven Design suggests, they have their own ubiquitous language. 
While an user is represented as an application user in authentication server, the same user is represented as a Trader in Item Trader API. 

Item Trader API is a client of auth server. 
The interactive app should log users in through auth server and than it can make request to Item Trader Api with obtained access token. These requests will be on behalf of the user. Item Trader Api uses auth server as Authority. Validates the token and
extracts the claims from it, which is only the User Id in this case. 

Interactive UI will use authorization code flow to obtain access_token which is authorized to access ItemTraderAPI scope. 
In Item Trader API perspective, if a request has valid access token, than the user should be authenticated already. And since no role based authorization is used, the user is capable to use all the endpoints, but only obtain related data with the user.
<details>

<br/>
<br/>

## One small challenge

There are some cases that two user wants to update same proposal at the same time. For that kind of situations, EntityFrameworkCore provides an optimistic locking mechanism. Since the app is operating on Status fields of records, i marked Status field as concurrency field. Doing this makes entity framework include this column as where clause for update and delete operations and if a change has occured since the entity is loaded in the context, it can't update or delete any data and determines a dirty data exists in the context. Throws an exception. These exceptions are catched in the system and converted to an appropriate message.



<!-- Part 2 -->
## Part 2

I used Azure as cloud platform. There are two app services running, one for Item Trader API and the other for Auth server. And also a SQL Server database exists again on Azure. 
CI/Cd Pipeline is built with github actions and terraform. My initial intent was building the pipeline with Azure DevOps Pipeline but due to some increased abuse on pipelines parallel tasks (coin miners they say), they require a request to use which will be resulted in 2 or 3 days. This makes me turn my way to Github Actions. 

Triggered with a push to master branch.
Steps for CI/CD pipeline are :
    
    * Restore dependencies
    * Build
    * Run Tests
    * Terraform init (initalizes workspace)
    * Terraform validate (validates the terraform file)
    * Terraform plan (creates a plan for resources to be provisioned or removed (Just a plan. No action happens at this step.))
    * Terraform apply (applies the created plan. Resources are provisioned at this step.) If doesn't exsit, terraform provisions the following at azure.
        * Resource group
        * SQL Server (creates the database on it as well)
        * Generates an app service plan
        * Creates Two app services running under this plan. 
            (One for Item Trader API and the other for Auth Server) 
            At this step connection strings and other sensitive data also create on app service.
             Those are not stored in source. 
             I used both Github secrets and Terraform Cloud User and Environment Variables.        
    * Publish Item Trader API
    * Publish Auth Server