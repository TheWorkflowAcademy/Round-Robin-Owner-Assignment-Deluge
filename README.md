# Round-Robin-Owner-Assignment-Deluge
Assign records (tasks) on a round robin pattern via a Deluge script in Zoho CRM.

## Core Idea
[Round robin assignment rule](https://help.zoho.com/portal/en/kb/crm/automate-business-processes/assignment-rules/articles/set-assignment-rules#Step_1_Enter_the_basic_details) is a cool feature in Zoho CRM but it has one major limitation - it does not work on the Activities module (Tasks/Meetings/Calls). The workaround is to perform the round robin assignment via a Deluge script. In this example, we will be doing a round robin assignment of specific tasks.

Here's an example scenario: When a Deal stage is updated to "Process Completed", you need to create a specific follow-up task assigned to the service agent team on a round robin pattern. In the same script that creates the task, we will write the script that subsquently updates the task owners based on the round robin algorithm.

## Configuration
* Connection scope needed:
  * ZohoCRM.settings.ALL

## Tutorial

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

### Make a List of Users with the Service Agent Profile and Get the Size
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
  * Get the previous task owner ID
	* A list begins from index 0 and by default, the records are sorted by descending order of created time.
	* So, if you `get(0)`, you'll get the most recent task. But what we need is the **previous task**, so, we `get(1)`.
  * Find the index of the Previous Owner ID in the Service Users list
  	* We already have the owner ID of the previous task.
  	* By using the `indexOf` function, we can get identify the index number of the previous task owner in the *serviceusers* list.
  * Use Modulus to set new Index value
  	* We now have the index number of the previous task owner. To get the index number of the next owner, we just need + 1.
  	* However, we can't just keep adding 1 as the number will exceed the list size.
  	* Here's where we use a mathematical operation called **modulus (%)**
  	* **Modulus** will in a way, sort of create a loop that keeps the index number running according to the list size.
  		* E.g. if the list size is 5, it will go 0,1,2,3,4,0,1,2,3,4...
  		* [Click here](https://blog.mattclemente.com/2019/07/12/modulus-operator-modulo-operation.html) to learn more about modular arithmetic.
  * Match the indexes to get the right new Owner ID
  	* We already have the next index number.
  	* To get the ID of the user, use the `.get()` function.
  	* At this point, you have the ID of the owner, ready for update!

```javascript
tasklist = zoho.crm.searchRecords("Tasks","(Subject:starts_with:" + "Complete Signed-App Process for" + ")");
//
if(size(tasklist) = 1)
{
	ownerid = serviceusers.get(0);
}
else
{
	// Get the previous task owner ID
	previousownerid = tasklist.get(1).get("Owner").get("id");
	info previousownerid;
	// Find the index of the Previous Owner ID in the Service Users list
	previousindex = serviceusers.indexOf(previousownerid);
	info previousindex;
	// Use Modulus to set new Index value
	nextindex = (previousindex + 1) % size;
	info nextindex;
	// Match the indexes to get the right new Owner ID.
	ownerid = serviceusers.get(nextindex);
	info ownerid;
}
```

### Create Task
Now that we have the *ownerid* ready, we can create the task assigned to the correct owner.
```javascript
newtask = Map();
newtask.put("Subject","Complete Signed-App Process for " + dealname);
newtask.put("Status","Not Started");
newtask.put("$se_module","Deals");
newtask.put("What_Id",dealid);
newtask.put("Due_Date",addBusinessDay(today,5));
newtask.put("Owner",ownerid);
createTask = zoho.crm.createRecord("Tasks",newtask);
info createTask;
taskid = createTask.get("id");
```
