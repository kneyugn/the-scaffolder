# The scaffolder

## Introduction

This project is built on top of [Liam Guilliver's tutorial](https://lgulliver.github.io/dynamically-generate-projects-with-github-templates-and-actions/). After running the below three curl commands successfully, you will have scaffolded a brand new Angular or .NET repository!

I updated versions of some action libaries to get it working again. The hope is that we can transfer these steps to run an internal organization's CLI to generate specific repositories such as new Angular, Nestjs projects that have specific standardizations required by internal tools.

- [Angular Example](https://github.com/kneyugn/angular-example/actions/runs/4717031895)
- [.NET Example](https://github.com/kneyugn/dotnet-example/actions/runs/4717043825)
- [React Example](https://github.com/kneyugn/react-example/actions/runs/4715445073) from templating (non-cli example)

I made some specific modifications to make this a UI first tool. The ideal flow would be:

`user registers repository with a UI form` --> `make repository from template` --> `add missing workflow permissions` --> `trigger workflow via repository_dispatch event`

---
## Prerequisite

### Generate personal access token:
To generate the personal access token, follow these steps below. I did not have much luck with fine grained PAT, so I went with the traditional tokens. 

```user menu -> settings -> developer settings -> personal access token -> generate new token```


### Environment variables to import into Insomnia/Postman.

```json
{
	"template_repo": "the-scaffolder",
	"template_owner": "kneyugn",
	"repository_dispatch_event": "on_scaffold_repo"
	"github_token": "<token>",
	"owner": "<your-github-user-name>",
	"repo": "<your-new-repo>",
	"scaffold_type": "<angular|dotnet|templating>",
	"user_name": "<your-github-user-name>",
	"user_email": "<your-email>",
}
```

**(In Insomnia, it is "CMD + E" to edit environment variables.)*

---

## How to scaffold

In general, the steps are as followed. 
1. API call to Github to make a template repository out of "kneyugn/the-scaffolder" template. Only worfklow files are carried over at this point.
1. API call to Github to add "write" access to permissions workflow endpoint.
1. API call to fire "request_dispatch" event with "templating" scaffold type and with the custom client_payload. This will trigger the workflow and call the CLI commands or do some templating before doing a git commit and push to the current repository.

### 1. Make the repository from template

```
curl --request POST \
  --url https://api.github.com/repos/{{ _.template_owner }}/{{ _.template_repo }}/generate \
  --header 'Accept: application/vnd.github+json' \
  --header 'Authorization: Bearer {{ _.github_token }}' \
  --header 'Content-Type: application/json' \
  --header 'X-GitHub-Api-Version: 2022-11-28' \
  --data '{
	"owner": "{{ _.owner }}",
	"name": "{{ _.repo }}",
	"description": "its scaffolded",
	"include_all_branches": false,
	"private": true
}'
```
*(In Insomnia, copy and paste entire thing into the input component)*

### 2. Enable workflow [permissions](https://docs.github.com/en/rest/actions/permissions?apiVersion=2022-11-28#set-default-workflow-permissions-for-a-repository) for the repo

```
curl --request PUT \
  --url https://api.github.com/repos/{{ _.owner }}/{{ _.repo }}/actions/permissions/workflow \
  --header 'Accept: application/vnd.github+json' \
  --header 'Authorization: Bearer {{ _.github_token }}' \
  --header 'X-GitHub-Api-Version: 2022-11-28' \
  --data '{"default_workflow_permissions":"write","can_approve_pull_request_reviews":true}'
```

*(In Insomnia, copy and paste entire thing into the input.)*
### 3. Dispatch [repository_dispatch](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#repository_dispatch) event 
Triggering `repository_dispatch` is preferred because it allows for the worfklow permissions step to be completed. Additionally, we can attach payload in the future that can be used by the CLI as arguments.

```
curl --request POST \
  --url https://api.github.com/repos/{{ _.owner }}/{{ _.repo }}/dispatches \
  --header 'Accept: application/vnd.github+json' \
  --header 'Authorization: Bearer {{ _.github_token }}' \
  --header 'X-GitHub-Api-Version: 2022-11-28' \
  --data '{
    "event_type": "{{ _.repository_dispatch_event }}",
    "client_payload": {
        "scaffold_type": "{{ _.scaffold_type }}",
        "user_name": "{{ _.user_name }}",
        "user_email": "{{ _.user_email }}",
	"owner": "{{ _.owner }}",
	"repo": "{{ _.repo }}"
    }
}'
```

## Template scaffolding

The template scaffolding method mimicks that Backstage is doing with software template scaffolding. Nunjucks is used under the hood to parse and replace template with variables.

The same process is used to scaffold from a template as it does from a CLI command. However, the templating needs a lot more client_payload data:

```
{
    "event_type": "{{ _.repository_dispatch_event }}", // {: event_type: "on_scaffold_repo"}
    "client_payload": {
        "scaffold_type": "{{ _.scaffold_type }}", // {scaffold_type: "templating"}
        "user_name": "{{ _.user_name }}", // <your-github-username>
        "user_email": "{{ _.user_email }}", // <your-email>
	"owner": "{{ _.owner }}", // <your-github-username>
	"repo": "{{ _.template_repo }}",// <your-new-repo>
	"templating_source": "{{ _.templating_source }}", // should be "kneyugn/dev-templates" or wherever the template lives
	"templating_definitions": { // the payload that nunjucks will use to replace variables. This part is dynamic for your template's needs
		"name": "{{ _.templating_definitions.name }}",
		"author": "{{ _.templating_definitions.author }}",
		"description": "{{ _.templating_definitions.description }}"
	}
    }
}
```

### How template scaffolding works

Templates are hosted in [dev-templates](https://github.com/kneyugn/dev-templates), but it can be any repository with the same structure.

The structure of a template should be the following. Note that the workflow relies on process-files.js to be in this specific location. process-files.js resides in the same template repository for independent testing and implementation. In most cases, process-files.js does not need to change.

```
- your-template-directory
- scripts
   - process-files.js
```

When a "repository_dispatch" event is called, the templating workflow runs the "build" job. It first clones from the "dev-templates" repo to get the nunjucks script and the corresponding template. The nunjucks templating engine runs through all of the files in the template directory and places the new file in the "output" directory. After, a git commit is made and changes are pushed to the current repository.
   
## Todo

- We do not need to move over all workflow files when a template is created. Instead the "kneyugn/the-scaffolder" template could be optimized to have only 1 workflow file. A new workflow files and a new repository_dispatch type called "on_pre_scaffolding" should be introduced. This workflow file, when executed, will pull down the matching workflow and commit this new file to the new repository. Then, we resume to send a "repository_dispatch" event of one of the following types "angular", "dotnet", or "templating" to execute the final workflow.
