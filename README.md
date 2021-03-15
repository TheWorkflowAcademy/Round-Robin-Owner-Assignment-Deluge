# Manual-Round-Robin-Assigner-Deluge
Assign records (tasks) on a round robin pattern via a Deluge script in Zoho CRM.

## Core Idea
[Round robin assignment rule](https://help.zoho.com/portal/en/kb/crm/automate-business-processes/assignment-rules/articles/set-assignment-rules#Step_1_Enter_the_basic_details) is a cool feature in Zoho CRM but it has one major limitation - it does not work on the Activities module (Tasks/Meetings/Calls). The workaround is to perform the round robin assignment via a Deluge script. In this example, we will be doing a round robin assignment of specific tasks.

Here's an example scenario: When a Deal stage is updated to "Process Completed", you need to create a specific follow-up task assigned to the service agent team on a round robin pattern. In the same script that creates the task, we will write the script that subsquently updates the task owners based on the round robin algorithm.

## Configuration
* Connection scope needed:
  * ZohoCRM.settings.ALL

## Tutorial
### Create Task
In the first part of the script, we create the task. Remember, we need a way to identify the task type - it could be by using the subject or a custom field. In this example, we're using the subject. Every task created will have a prefix of "Complete Signed-App Process for" as the subject. 
*Note: The script to create task is not shown here*

### Get a List of Active Users
Use a Zoho CRM API call to get a list of Active Users in the system.

```javascript
userslist = invokeurl
[
	url :"https://www.zohoapis.com/crm/v2/users"
	type :GET
	parameters:{"type":"ActiveUsers"}
	connection:"zohocrm_oauth_connection"
];
users = userslist.get("users");
```

### Make a List of Users with the Service Agent Profile and get the Size
Iterate through the *userlist* and create a new list consisting of only Service Agents (change this to whatever user profile you need).

```javascript
serviceusers = List();
for each  u in users
{
	if(u.get("profile").get("name").contains("Service Agent"))
	{
		serviceusers.add(u.get("id"));
	}
}
size = size(serviceusers);
info size;
```

### Assign the New Owner ID on a Round Robin Pattern
* We use the `searchRecords` function to get the *tasklist* of all tasks with subject that starts with the specific prefix.
* If the size of the *tasklist* is 1, it means that it is the first iteration (there is no previous task).
  * Here, we assign the owner by simply getting the first index of the *serviceusers* list.
* If the *tasklist* size is more than 1,
  * Get the previous task owner ID (get

```javascript
tasklist = zoho.crm.searchRecords("Tasks","(Subject:starts_with:" + "Complete Signed-App Process for" + ")");
//
if(size(tasklist) = 1)
{
	newownerid = serviceusers.get(0);
}
else
{
	previousownerid = tasklist.get(1).get("Owner").get("id");
	info previousownerid;
	// Find the index of the Previous Owner ID in the Service Users list
	previousindex = serviceusers.indexOf(previousownerid);
	info previousindex;
	// Use Modulus to set new Index value
	nextindex = (previousindex + 1) % size;
	info nextindex;
	// Match the indexes to get the right new Owner ID.
	newownerid = serviceusers.get(nextindex);
	info newownerid;
}
```
