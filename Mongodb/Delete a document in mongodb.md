## Delete a document in mongodb using mongo shell

1. Use <code>mongo</code> to enter the mongo cli shell on linux

2. <code>show dbs</code> query will list all the available databases. To switch to a particular db use cmd <code>use dbname</code>

3.  To find a document with particular json field in a tenants collection use <code>db.tenants.find({tenantCode: "OMNYK_IND-001" })</code> query

4.  To delete the document with particular json field in a tenants collection use <code>db.tenants.remove({tenantId: "OMNYK_IND-001" })</code> query

5.  Verify the deletion by again running the find query <code>db.tenants.find({tenantCode: "OMNYK_IND-001" })</code>
6.  Add or update fields to a document in a collection `db.users.update( { "id": "e6f32a98-5c53-42ac-b5f3-9a7e0658fb70" }, { $set: { "emergencyContact" : "+91 9448908617", "emergencyEmail" : "pintu@omnyk.com"  }  } )`