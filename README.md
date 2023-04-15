# The scaffolder

## Introduction
This project is built on top of [Liam Guilliver's tutorial](https://lgulliver.github.io/dynamically-generate-projects-with-github-templates-and-actions/). After running the below three curl commands successfully, you will have scaffolded a brand new Angular or .NET repository!

I updated versions of some action libaries to get it working again. The hope is that we can transfer these steps to run an internal organization's CLI to generate specific repositories such as new Angular, Nestjs projects that have specific standardizations required by internal tools.

- [Angular Example](https://github.com/kneyugn/angular-scaffold-example/actions/runs/4709745047)
- [.NET Example](https://github.com/kneyugn/dotnet-scaffold-example/actions/runs/4709736617)

I made some specific modifications to make this a UI first tool. The ideal flow would be:

`user registers repository with a UI form` --> `make repository from template` --> `add missing workflow permissions` --> `trigger workflow via repository_dispatch event`

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
	"scaffold_type": "<angular|dotnet>",
	"user_name": "<your-github-user-name>",
	"user_email": "<your-email>",
}
```

**(In Insomnia, it is "CMD + E" to edit environment variables.)*

### Make the repository from template

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

### Enable workflow [permissions](https://docs.github.com/en/rest/actions/permissions?apiVersion=2022-11-28#set-default-workflow-permissions-for-a-repository) for the repo

```
curl --request PUT \
  --url https://api.github.com/repos/{{ _.owner }}/{{ _.repo }}/actions/permissions/workflow \
  --header 'Accept: application/vnd.github+json' \
  --header 'Authorization: Bearer {{ _.github_token }}' \
  --header 'X-GitHub-Api-Version: 2022-11-28' \
  --data '{"default_workflow_permissions":"write","can_approve_pull_request_reviews":true}'
```

*(In Insomnia, copy and paste entire thing into the input.)*
### Dispatch [repository_dispatch](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#repository_dispatch) event 
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

## Todo
The scaffolded repo needs to remove the .angular and .dotnet workflow files.
