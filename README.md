# On-Premises Setup

## Infrastructure Requirements
### 1. Two domain names

* For Data Collection Engine (like growlyticsdc.<company_name>.com )
* Company applications will refer this domain for sending data to Growlytics.
* For Growlytics Dashboard (like growlytics.<company_name>.com)
* User will use Growlytics Dashboard to see company insights. 

### 2. MongoDB 

* Growlytics use MongoDb as storage engine.
* Requirements: Min ```2 GB``` RAM, Min ```3 GB``` Storage.

### 3. One Linux server

* For Hosting Growlytics Data collection engine & reporting engine.
* Requirements: Min ```4 GB``` RAM, Min ```3 GB``` Storage (t2.medium EC2 instance will work).

### 4. Email Account (for sending email notifications)

* To send event notifications to different stakeholders

### 5. SSL certificate (if website is using ssl)

* If the website being integrated with Growlytics is using SSL, then also SSL certificate will be required.

### 6. AWS S3 Bucket (to save image snapshots)

* Growlytics captures browser snapshots for browser events.
* And these event snapshots are stored in S3.

### 7. MySql DB (for storing Project setting)

* Growlytics uses Mysql for storing company settings, project settings, user list etc.

### 8.   Redis DB

* Growlytics use Redis for session management.
* Requirements: 

### 9A. File System 

* Growlytics also provides a alternative store for Session Management.
* Requirements: A folder with read & write permission at location ```/var/growlytics/web_sessions```.

### 9B. File System

* Growlytics also provides a alternative store for Logger Management.
* Requirements: A folder with read & write permission at location ```/var/growlytics/browser_log``` for browser logs and 
```/var/growlytics/server_log``` for server logs.

---

## Node.js Integration

### Step 1: Installation
* Add Growlytics server plugin to your express application.
<a href="https://gist.github.com/satyampatro/ad185a117855aeb01f9ed97055731f28" download>growlytics.server.min.js - 8KB</a>

### Step 2: Require growlytics in your app & add your api key
```
var Growlytics = require('<path_to_growlytics.server.min.js>');
var growlytics = new Growlytics({
                    projectCode: '<project_code>', 
                    apiKey: '<api_key>'
                 });
```
                 
> **Make sure you are replacing <project_code>,<api_key>, with your project code and project api key.**

### Step 3: Register Growlytics middleware

* To ensure that apis are captured by growlytics, add the requestHandler middleware to your app as the first middleware.
```app.use(growlytics.requestHandler());```

### Step 4: Register Growlytics middleware for error reporting

* To ensure that unhandled errors are captured by growlytics, add the errorHandler middleware to your app as the last middleware.
```app.use(growlytics.errorHandler());```

### Step 5: Write logs to growlytics
* Writing logs to growlytics is a bit tricky. Growlytics maintains logs seperate for request. Hence, logger instance is only available from the express's request object.

*If you have registered Growlytics request handling middleware(Step 2), then you can use ```req.getGrowlyticsLogger()``` method to get logger instance. Here ```req``` is express request object.*

```var logger = req.getGrowlyticsLogger();```
*After getting Growlytics logger object, you can write logs at debug, info, warning and error level.*
```
req.getGrowlyticsLogger().info("message", { property: "value" });
req.getGrowlyticsLogger().debug("message", { property: "value" });
req.getGrowlyticsLogger().warning("message", { property: "value" });
req.getGrowlyticsLogger().error("message", { property: "value" });
```

Complete example is given below:Index.js
```
app.post("/login", function (req, res, next) {      
var userName = req.body.userName;   
var companyCode = req.body.companyCode;     
//Write Logs   
var logger = req.getGrowlyticsLogger();   
logger.info("New Login Request", {       
    'userName': userName,      
    'companyCode': companyCode   
});    
req.send('Growlytics Logging Test Complete.');});
```

---

## Front-end Integration

### Step 1: Installation
* Add the file to your client project.
<a href="https://gist.github.com/satyampatro/48f61bd334364c6ff1faf0ed860b5496" download>growlytics.server.min.js - 8KB</a>

### Step 2: Add Javascript SDK to your web app.

* To use the JavaScript SDK, simply copy-paste the following code into the <head> of any page you'd like to monitor. Add it before all other <script> tags to ensure that errors in those scripts are reported to Growlytics.

````
<script>    
   //Growlytics Client Configuration
    window['_grw_project_code'] = '<project_code>'; 
    window['_grw_api_key'] = '<project_api_key>';
    window['_grw_use_domain'] = 'https://growlytics.dc.<company_name>.com';
    window['_grw_use_port'] = '<dc_server_port>';
    window['_grw_namespace'] = 'Growlytics';
    window['_grw_disabled'] = false; // make it true to disable the plugin

    var jsElm = document.createElement("script");
    jsElm.type = "application/javascript";
    jsElm.src = "<path_to_growlytics.client.min.js>";
    document.body.appendChild(jsElm);
</script>
````
> **Make sure you are replacing <project_code>,<api_key>, <path_to_growlytics.client.min.js>with your project code, project api key and path to growlytics client plugin respectively.**

---

## MySQL Setup for Company Projects

### Step 1: Run this below queries to create a table schema as required to work with our growlytics application.
```
// create company table
CREATE TABLE `company` (
	  `company_code` varchar(20) NOT NULL DEFAULT '',
	  `subscription` varchar(255) DEFAULT NULL,
	  `company_name` varchar(255) NOT NULL,
	  PRIMARY KEY (`company_code`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

//create user table
CREATE TABLE `user` (
	  `id` varchar(20) NOT NULL DEFAULT '',
	  `first_name` varchar(255) NOT NULL,
	  `last_name` varchar(255) NOT NULL,
	  `email` varchar(255) DEFAULT NULL,
	  `password` varchar(255) NOT NULL,
	  `company` varchar(20) NOT NULL DEFAULT '',
	  `role` enum('user','admin') NOT NULL DEFAULT 'user',
	  `token` varchar(45) DEFAULT NULL,
	  `token_timestamp` varchar(45) DEFAULT NULL,
	  PRIMARY KEY (`id`),
	  UNIQUE KEY `uniq_email_company` (`email`,`company`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

// create user table
CREATE TABLE `project` (
	  `id` varchar(20) NOT NULL,
	  `code` varchar(20) NOT NULL,
	  `company` varchar(20) NOT NULL,
	  `description` varchar(1000) DEFAULT NULL,
	  `name` varchar(255) NOT NULL,
	  `api_key` varchar(255) NOT NULL,
	  `db_location` varchar(1000) DEFAULT NULL,
	  `db_name` varchar(1000) DEFAULT NULL,
	  `api_retention_days` int(11) NOT NULL DEFAULT '30',
	  `session_retention_days` int(11) DEFAULT '30',
	  PRIMARY KEY (`id`),
	  UNIQUE KEY `code` (`code`),
	  KEY `fk_project_company_idx` (`company`),
	  CONSTRAINT `fk_project_company` FOREIGN KEY (`company`) REFERENCES `company` (`company_code`) ON DELETE NO ACTION ON UPDATE NO ACTION
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

// create user project map
CREATE TABLE `user_project_map` (
    `user_id` varchar(20) NOT NULL DEFAULT '',
    `project_id` varchar(20) NOT NULL DEFAULT '',
    PRIMARY KEY (`user_id`,`project_id`),
    KEY `fk_user_project_map_user_code_idx` (`user_id`),
    KEY `fk_user_project_map_project_code_idx` (`project_id`),
    CONSTRAINT `fk_user_project_map_project_id` FOREIGN KEY (`project_id`) REFERENCES `project` (`id`) ON DELETE NO ACTION ON UPDATE NO ACTION,
    CONSTRAINT `fk_user_project_map_user_id` FOREIGN KEY (`user_id`) REFERENCES `user` (`id`) ON DELETE NO ACTION ON UPDATE NO ACTION
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```
### Step 2: Insert a company & project in company & project table respectively to get started with project setup.

```
// insert a reow in company table
INSERT INTO `company` (`company_code`, `subscription`, `company_name`) 
VALUES ('<company_code>', '<subscription>', '<company_name>');

// insert a row in project table
INSERT INTO `project` (`id`, `code`, `company`, `description`, `name`, `api_key`, `db_location`, `db_name`, `api_retention_days`, `session_retention_days`)
VALUES ('<random_string>', '<project_code>', '<fk_company_code>', '<project_description>', '<project_name>', '<api_key>', '<mongodb_location_string>', '<db_name>', '<api_retenstion_days>', '<session_retention_days>');
```
### Step 3: Create a user account at  growlytics.<company_name>.com/register.

### Step 4: Map your user account id with project id by inserting a row in table user_project_map.
```
// insert a row to map user an project
INSERT INTO `user_project_map` (`user_id`, `project_id`) 
VALUES ('<fk_user_id>', '<fk_project_id>');
```
**Setup completed & now login with your credentials and add new members to your project or can also integrate growlytics to company's other projects and then add that project settings to MySQL.**

