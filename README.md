
Granular Permissions API
===

### The Problem
It's hard to parse granular permissions for a tool as the API filters only by user.
Let's have the Permissions endpoint take in a tool as param and return permissions specific to that tool. We can also allow a mroe granular checking of permissions, as in by module, e.g. permission for Locations (can we create on the fly?).

### Why this is a Problem & Existing Solution
Current API: https://developers.procore.com/reference/permissions

In iOS for example, we have to do all this:

```
   private func fetchGranularPermissions(completion: @escaping ((Bool)->Void)) {
        let path = "/vapid/settings/permissions"
        let query = ["project_id": projectId]
        PROHttp.get(path, query: query) { result in
            switch result {
            case .failure(let error):
                Log.error("[DailyLogsViewModel] failed to fetch granular permissions: \(error.localizedDescription)")
            case .success:
                do {
                    let helper = try Json.helper(from: result)
                    try helper.forKey("tools", { tools in
                        try tools.forEach({ json in
                            if let name: String = try json.value(forKey: "name"), name == "daily_log" {
                                let permissions: [Any] = try json.value(forKey: "permitted_actions")
                                permissions.forEach({ action in
                                    if let actionObject = action as? [String: Any], let name = actionObject["action_name"] as? String {
                                        if name == "log_restrictively" {
                                            completion(true)
                                        }
                                    }
                                })
                            }
                        })

                        completion(false)
                    })
                } catch(let error) {
                    Log.error("[DailyLogsViewModel] error parsing json: \(error.localizedDescription)")
                    completion(false)
                }
            }
        }
    }
```


...to get a single boolean for whether the user has granular permissions.
It's a lot of code for not a lot of logic.

We use this, for example, when in the view controller, we want to decide which log modules to show. Other tools would also probably benefit from a clean API to get permissions for a specific tool; it'd be easy logic for views to handle hiding or showing certain UI elements.

### Proposal

We should send a simple JSON response with all the permissions of a user given its ID and the tool, or the module.

Sample request: ```GET /apiv3/settings/permissions?project_id=420814&tool_name=daily_log```

Sample response:
```
{
  "permissions": {
    "base" : "read_only",
    "collaborator" : "true"
  }
}
```


Sample request: ```GET /apiv3/settings/permissions?project_id=619042&module_name=location_picker```

Sample response:
```
{
  "permissions": {
    "canCreate" : "true",
    "canCreateOnTheFly" : "false"
  }
}
```

Thus, the client code can be as simple as (pseudocode): 
```
let path = "/vapid/settings/permissions"
let query = ["project_id": projectId, "tool_name": "daily_log"]
PROHttp.get(path, query: query) { result in
   if(result == .failure) return
   let helper = try Json.helper(from: result)
   if(helper.forKey("collaborator")=="true")
      completion(true)
   else 
      completion(false)
}
```

### Others' solution
```
"customData": {
   “permissions”:
       “crew_quarters”: “9-3601”,
       "lock_override”: “all”,
       "command_bridge”: {
          “type”: “vessel:bridge”,
          “identifier”: “NCC-1701-D”,
          “action”: “lockout”,
          "control_key”: "173467321476789764376",
       }
    }
```
from https://stormpath.com/blog/fine-grained-permissions-with-customdata

Given the user, it'll return the permissions for that user's tool portfolio as deep objects so we can easily key into the tool we want. This is more usable than our current API that returns nested arrays and objects where we have to do for loops to find an item with a certain key-value.


### Potential drawbacks/pitfalls
Maybe the endpoint will be hit more as it is convenient, encouraging clients to not have one call for all permissions and handle individual ones locally... but that's like saying, "Oh our tool is too good and we're worried people are gonna use it too much."

### Followups

Potentially restructure the current API to return object keyed by tool_name rather than an array of tools that has to be iterated through.
