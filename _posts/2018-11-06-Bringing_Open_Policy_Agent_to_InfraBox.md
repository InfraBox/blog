---
layout: post
title: Bringing the Open Policy Agent to InfraBox
---

With the latest release of InfraBox the Open Policy Agent (OPA) is used for authorization checks in the API.

The Open Policy Agent is a dedicated service for policy handling. Instead than in previous InfraBox releases those checks are not performed in the InfraBox API itself. Instead, the InfraBox API *asks* the OPA service before executing any request whether the client is authorized to perform the requested action in the respective situation.

## Authorization procedure

When a request reaches the InfraBox API, the API sends a query to the OPA service before executing any action. Included in that query is context data including the HTTP method, the API path and - if available - any authentication data like a user or a project token.

OPA evaluates that request through policy files we uploaded to OPA before. These files contain rules in the OPA-specific and by Datalog inspired rego language.

![Communication between Client, InfraBox API and Open Policy Agent]({{ site.baseurl
}}/images/2018-11-06-OpenPolicyAgent/opa_communication.png "Communication between Client, InfraBox API and Open Policy Agent")

## Context replication

For evaluating some requests the OPA service requires additional data that is not provided by the input. An example for that is a user accessing a project: A user may see a project if the project is either public or the user is a collaborator of that project. The OPA service needs to have access to a list of projects and which o#f them are public, and a list of collaborators per project.

To achieve that we replicate this data: Always when this data is changed and additionally in a time interval set in the configuration an updated version is pushed to OPA.

## Writing policy files

Let's have a look on an example policy file. This file restricts the read-access for private projects to collaborators and grants everyone access if the project is public. By default, we deny any requests, and define rules when to allow a request.

```javascript
import data.infrabox.collaborators.collaborators
import data.infrabox.projects.projects

project_public(project){
    projects[i].id = project
    projects[i].public = true
}

project_collaborator([user, project]) {
    collaborators[i].project_id = project
    collaborators[i].user_id = user
}

allow {
    api.method = "GET"
    api.path = ["api", "v1", "projects", project]
    api.token.type = "user"
    project_collaborator([api.token.user.id, project])
}
allow {
    api.method = "GET"
    api.path = ["api", "v1", "projects", project]
    project_public(project)
}
```

First we import our data that we have replicated to OPA. For not having to rewrite the check whether a project is public we define a function `project_public` which returns `true` if there is a project `i` that has the given project id and is public. We do the same for the check if a user is collaborator of a project: If there is a collaborator record `i` where the user and the project id match to the given ids the function `project_collaborator` returns `true`.

After these functions there are multiple allow statements which means that the request is allowed if one of these statements evaluates to `true`.

## Testing OPA

If you want to test InfraBox's OPA service on its own you can start it up by running `./ib.py services start opa` in the InfraBox root folder. The defined policies are deposited in the folder `src/openpolicyagent/policies/`.

## Conclusion

The Open Policy Agent enables applications to seperate the application logic from the authorization handling. It simplifies making policies configurable and exchangeable. Especially in multi-server environments and authorization checks on different application layers pays off. A possible dealbreaker however might be the need to provide OPA with context data. Besides replication there are also other ways to deal with that, like built-in functions or Partial Evaluation, but those come with either performance drawbacks or the necessity to add additional application logic.